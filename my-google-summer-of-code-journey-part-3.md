# Time for a technical deep dive into my project.

In the previous blog, I gave an **overview** of my journey, mostly keeping it **noob-friendly** and showing what a realistic journey from the preparation, ideation, acceptance, to the completion looks like.

This would be my last article about my GSoC journey. In this article, I’ll be going down into the **technical** **depths** of my project. You may or may not find it interesting, but if OS, kernels, concurrency, memory management (and **mismanagement** too) tickle your brain, do stick around. I’ll post more technical and non-technical content in the upcoming blogs.

## Overview of my project.

As mentioned previously, I completed GSoC’25 in The FreeBSD Project. **FreeBSD**, as you might know, is a **Unix-like** operating system known for its stability, performance, and **security**. My project was medium-sized (175 hours), made of two small-sized projects, namely (i) `mac_do(4)` improvements and (ii) `mdo(1)` improvements.

## mac\_do(4) improvements

`mac_do(4)` is a kernel security module that can enable **controlled process credentials transitions**, such as changing the user IDs or group IDs to particular values. Processes that make such requests and are authorized `mac_do(4)` do not need to be root or spawned from an executable with the **setuid** bit. It overcomes the fundamental security issues of programs like `sudo` or `doas`.

I had two objectives for this module:

1. Allow `mac_do(4)` to support a list of **executables** **authorized** to request credential transitions, and enable the ability to configure this list in `jail(8)`s.
    
2. Allow `mac_do(4)` to support authorizing credential transitions outside the `setcred(2)` system call, i.e., implementing support for traditional system calls like `setuid(2)`, `setgid(2)`, `setgroups(2)`, etc.
    

### Objective 1: Per-jail configurable list for executable paths

One major drawback of the original `mac_do(4)` module was that it was **hardcoded** to only allow processes that spawned from `/usr/bin/mdo`. Any other application wishing to take advantage of the credential transition capabilities would be **out of luck** due to this rigidity. As with the credential transition rules, it was my responsibility to make this configurable on a per-jail basis.

The existing infrastructure was built around managing `rules` - the credential transition specifications. However, I needed to store and manage both **rules** and **executable** **paths** together. The original structure was simple:

```c
struct rules {
    char        string[MAC_RULE_STRING_LEN];
    struct rulehead head;
    volatile u_int  use_count __aligned(CACHE_LINE_SIZE);
};
```

This needed to evolve into something more comprehensive that could handle multiple types of **configuration** data while maintaining the same **lifetime** **management** and **concurrency** guarantees.

**Introducing struct conf: A Unified Configuration Container**  
The solution was to create a new `struct conf` that would encapsulate both rules and executable paths:

```c
struct exec_paths {
    char exec_paths_str[EXEC_PATHS_MAXLEN];
    char exec_paths[MAX_EXEC_PATHS][PATH_MAX];
    int exec_path_count;
};

struct conf {
    struct rules rules;
    struct exec_paths exec_paths;
    volatile u_int use_count __aligned(CACHE_LINE_SIZE);
};
```

This design adds the new `exec_paths` functionality while preserving the original `struct rules`. In order to ensure **atomic** updates of both rules and executable paths, the **reference** **counting** mechanism that was previously at the rules level now functions at the configuration level.

**Memory Management and Lifetime Challenges**  
One of the trickiest aspects was migrating all the lifetime management logic from `rules` to `conf`. The original code had functions like:

```c
static struct rules *alloc_rules(void)
static void toast_rules(struct rules *const rules)  
static void hold_rules(struct rules *const rules)
static void drop_rules(struct rules *const rules)
```

These all needed to be reimplemented for the new configuration structure:

```c
static struct conf *alloc_conf(void)
static void drop_conf(struct conf *const conf)
{
    if (refcount_release(&conf->use_count)) {
        toast_rules(&conf->rules);  // Clean up embedded rules
        free(conf, M_MAC_DO);       // Free the container
    }
}
```

The key insight was that `struct conf` now owns the **lifetime** of the **embedded** `struct rules`, so the cleanup process needed to handle the nested structure properly.

**Deep vs Shallow Copying: A Critical Distinction**  
I ran into a minor but significant copying problem when implementing configuration inheritance and updates. The dynamically allocated arrays (`uids` and `gids`) in the `struct rules` are not easily copied using `bcopy()`. I had to put deep cloning functions into practice:

```c
static void
clone_rules(struct rules *dst, struct rules *const src)
{
    // Copy the basic structure
    strlcpy(dst->string, src->string, sizeof(dst->string));
    STAILQ_INIT(&dst->head);
    
    // Deep copy each rule with its dynamic arrays
    STAILQ_FOREACH(src_rule, &src->head, r_entries) {
        dst_rule = malloc(sizeof(*dst_rule), M_MAC_DO, M_WAITOK | M_ZERO);
        bcopy(src_rule, dst_rule, sizeof(*dst_rule));
        
        if (src_rule->uids_nb > 0) {
            dst_rule->uids = malloc(sizeof(*dst_rule->uids) * src_rule->uids_nb,
                M_MAC_DO, M_WAITOK);
            bcopy(src_rule->uids, dst_rule->uids,
                sizeof(*dst_rule->uids) * src_rule->uids_nb);
        }
        // Similar for gids...
    }
}
```

This was required because jail configuration inheritance meant that a jail had to inherit from its parent, but as an **independent** **copy** rather than a **shared** **reference**, when it didn't specify its own rules or paths.

**Concurrency and Prison Lock Management**  
The kernel environment requires careful attention to **locking** and **concurrency**. The existing infrastructure used **prison locks** to protect **jail-specific data**, and I had to ensure this remained consistent:

```c
static struct conf *
find_conf(struct prison *const pr, struct prison **const aprp)
{
    struct prison *cpr, *ppr;
    struct conf *conf;
    cpr = pr;
    for (;;) {
        prison_lock(cpr);
        conf = osd_jail_get(cpr, osd_jail_slot);
        if (conf != NULL)
            break;
        prison_unlock(cpr);
        ppr = cpr->pr_parent;
        MPASS(ppr != NULL); // prison0 always has config
        cpr = ppr;
    }
    *aprp = cpr;
    return (conf);
}
```

The caller receives both the configuration and the prison that holds it, and must **unlock** that prison when done. This pattern ensures that the configuration remains valid for the duration of use while preventing deadlocks.

**Jail Parameter Integration**  
Adding the new executable paths required extending the **jail** **parameter** system. This meant implementing new sysctl handlers and jail management hooks:

```c
static int
mac_do_sysctl_exec_paths(SYSCTL_HANDLER_ARGS)
{
    char *const buf = malloc(EXEC_PATHS_MAXLEN, M_MAC_DO, M_WAITOK);
    struct conf *conf;
    struct prison *pr;
    
    conf = find_conf(td_pr, &pr);
    strlcpy(buf, conf->exec_paths.exec_paths_str, EXEC_PATHS_MAXLEN);
    prison_unlock(pr);
    
    // Handle updates...
}

SYSCTL_PROC(_security_mac_do, OID_AUTO, exec_paths,
    CTLTYPE_STRING | CTLFLAG_RW | CTLFLAG_PRISON | CTLFLAG_MPSAFE,
    0, 0, mac_do_sysctl_exec_paths, "A",
    "Colon-separated list of allowed executables");
```

**Transforming check\_proc(): From Static to Dynamic**  
The original `check_proc()` function was trivial; just a hardcoded string comparison:

```c
// Original version
error = strcmp(path, "/usr/bin/mdo") == 0 ? 0 : EPERM;
```

The new version needed to dynamically check against the jail's configured executable paths:

```c
static int
check_proc(void)
{
    char *path, *to_free;
    int error = EPERM;
    
    if (vn_fullpath(curproc->p_textvp, &path, &to_free) != 0)
        return (EPERM);
    
    struct conf *conf;
    struct prison *pr;
    conf = find_conf(curproc->p_ucred->cr_prison, &pr);
    
    for (int i = 0; i < conf->exec_paths.exec_path_count; i++) {
        if (strcmp(conf->exec_paths.exec_paths[i], path) == 0) {
            error = 0;
            break;
        }
    }
    
    prison_unlock(pr);
    free(to_free, M_TEMP);
    return (error);
}
```

This change enables per-jail customization of authorized executables while maintaining the security model.

**OSD Integration and Default Configuration**  
The jail lifecycle hooks had to be updated to accommodate the new configuration structure for the **Object-Specific Data (OSD)** framework integration. The default configuration needed to provide backward compatibility:

```c
static void
set_default_conf(struct prison *const pr)
{
    struct conf *const conf = alloc_conf();
    // Maintain backward compatibility
    strlcpy(conf->exec_paths.exec_paths_str, "/usr/bin/mdo", EXEC_PATHS_MAXLEN);
    strlcpy(conf->exec_paths.exec_paths[0], "/usr/bin/mdo", PATH_MAX);
    conf->exec_paths.exec_path_count = 1;
    set_conf(pr, conf);
}
```

**The Result: Flexible, Per-Jail Configuration**  
`mac_do(4)` was successfully transformed from a rigid, hardcoded system to one that is completely configurable. Now, administrators can specify a list of executable paths for various jails:

```bash
# System-wide configuration
sysctl security.mac.do.exec_paths="/usr/bin/mdo:/usr/local/bin/custom_tool"
# Per-jail configuration  
jail -c name=myjail mac.do.exec_paths="/usr/bin/mdo:/usr/local/bin/jail_specific_tool"
```

### Objective 2: Implementing support for traditional credentials changing syscalls

One major drawback of the original `mac_do(4)` implementation was that it could only approve credential transitions that were requested using the specific `setcred(2)` system call. Traditional **POSIX** system calls such as `setuid(2)`, `setgid(2)`, and `setgroups(2)` are usually used in real-world applications, even though `setcred(2)` offers complete credential management. In order to intercept and authorize these common credential-changing functions, I had to extend `mac_do(4)`.

**The Infrastructure Challenge**  
The existing code already had a sophisticated infrastructure for handling `setcred(2)` through the MAC framework:

```c
struct mac_do_data_header {
    size_t allocated_size;
    size_t size;
    int priv;           // Privilege identifier
    struct conf *conf;  // Configuration to apply
};

struct mac_do_setcred_data {
    struct mac_do_data_header hdr;
    const struct ucred *new_cred;
    u_int setcred_flags;
};
```

This infrastructure uses **per-thread Object-Specific Data (OSD)** to pass information between **MAC** hooks safely, handling **concurrency** and memory management automatically. The challenge was extending this pattern to handle multiple different system calls.

**Creating Specialized Data Structures**  
I created dedicated data structures for different categories of system calls:

```c
struct mac_do_setuid_data {
    struct mac_do_data_header hdr;
    uid_t target_uid;
};

struct mac_do_setgid_data {
    struct mac_do_data_header hdr;
    gid_t target_gid;
};

struct mac_do_setgroups_data {
    struct mac_do_data_header hdr;
    int ngroups;
    gid_t *groups;
};
```

Each structure shares the common header but carries syscall-specific data needed for authorization decisions.

**Expanding the Privilege Grant Logic**  
The original `mac_do_priv_grant()` function handled only `PRIV_CRED_SETCRED`. I extended it to handle all the new privilege types:

```c
static int
mac_do_priv_grant(struct ucred *cred, int priv)
{
    switch (priv) {
        case PRIV_CRED_SETCRED: {
            // Original setcred logic
            break;
        }
        
        case PRIV_CRED_SETUID:
        case PRIV_CRED_SETEUID:
        case PRIV_CRED_SETREUID:
        case PRIV_CRED_SETRESUID: {
            struct mac_do_setuid_data *data = fetch_data();
            // Check rules against target UID
            STAILQ_FOREACH(rule, &rules->head, r_entries)
                if (rule_applies(rule, cred)) {
                    error = rule_grant_user(rule, cred, data->target_uid);
                    if (error != EPERM) break;
                }
            return (error);
        }
        
        case PRIV_CRED_SETGROUPS: {
            struct mac_do_setgroups_data *data = fetch_data();
            // Check rules against target groups
            STAILQ_FOREACH(rule, &rules->head, r_entries)
                if (rule_applies(rule, cred)) {
                    error = rule_grant_setgroups(rule, cred, 
                            data->ngroups, data->groups);
                    if (error != EPERM) break;
                }
            return (error);
        }
        // Similar cases for GID syscalls...
    }
}
```

**Implementing the Hook Triplets**  
Each system call requires three MAC hooks: enter, check, and exit. I implemented these for all nine syscalls (`setuid`, `seteuid`, `setreuid`, `setresuid`, `setgid`, `setegid`, `setregid`, `setresgid`, `setgroups`). The pattern is consistent:

```c
static void
mac_do_setuid_enter(void)
{
    // Set up per-thread data if authorized executable
    if (do_enabled == 0 || check_proc() != 0)
        return;
        
    conf = find_conf(curproc->p_ucred->cr_prison, &pr);
    hold_conf(conf);
    
    data = alloc_data(data, sizeof(*data));
    set_data_header(data, sizeof(*data), PRIV_CRED_SETUID, conf);
}

static int
mac_do_check_setuid(struct ucred *cred, uid_t uid)
{
    // Capture the target UID for privilege checking
    data->target_uid = uid;
    return (0);
}

static void
mac_do_setuid_exit(void)
{
    // Clean up per-thread data
    clear_data(data);
}
```

**Handling Complex Parameter Logic**  
Some syscalls like `setreuid(2)` and `setresuid(2)` can take multiple UIDs where `-1` means "don't change". I had to implement logic to determine the effective target:

```c
static int
mac_do_check_setreuid(struct ucred *cred, uid_t ruid, uid_t euid)
{
    if (euid != (uid_t)-1)
        data->target_uid = euid;      // Effective UID takes priority
    else if (ruid != (uid_t)-1)
        data->target_uid = ruid;      // Fall back to real UID
    else
        data->target_uid = cred->cr_uid;  // No change requested
    return (0);
}
```

**Adapting Group Authorization Logic**  
The `setgroups(2)` syscall required special attention since it operates differently from `setcred(2)`. I created `rule_grant_setgroups()` that works directly with the group array:

```c
static int
rule_grant_setgroups(const struct rule *const rule,
    const struct ucred *const old_cred, const int ngroups, 
    const gid_t *const groups)
{
    // Check each group in the new groups array
    for (int new_idx = 0; new_idx < ngroups; ++new_idx) {
        const gid_t gid = groups[new_idx];
        bool may_accept = false;
        
        // Apply same rule logic as setcred supplementary groups
        // but work directly with the groups array
    }
    return (0);
}
```

**MAC Framework Integration**  
Beyond the module changes, this required extending the MAC framework itself. I added new MAC policy entry points in the kernel:

```c
// In mac_policy.h
typedef void (*mpo_cred_setuid_enter_t)(void);
typedef int (*mpo_cred_check_setuid_t)(struct ucred *cred, uid_t uid);
typedef void (*mpo_cred_setuid_exit_t)(void);
// ... and similar for all other syscalls

// In kern_prot.c - example for setuid()
MAC_CRED_SETUID_ENTER();
error = priv_check_cred(oldcred, PRIV_CRED_SETUID);  
MAC_CRED_CHECK_SETUID(newcred, uid);
if (error) {
    MAC_CRED_SETUID_EXIT();
    return (error);
}
// ... perform the actual credential change
MAC_CRED_SETUID_EXIT();
```

Finally, I registered all the new hooks in the MAC policy operations structure:

```c
static struct mac_policy_ops do_ops = {
    // Original hooks
    .mpo_cred_setcred_enter = mac_do_setcred_enter,
    .mpo_cred_check_setcred = mac_do_check_setcred,
    .mpo_cred_setcred_exit = mac_do_setcred_exit,
    
    // New UID syscall hooks
    .mpo_cred_setuid_enter = mac_do_setuid_enter,
    .mpo_cred_check_setuid = mac_do_check_setuid,
    .mpo_cred_setuid_exit = mac_do_setuid_exit,
    
    .mpo_cred_seteuid_enter = mac_do_seteuid_enter,
    .mpo_cred_check_seteuid = mac_do_check_seteuid,
    .mpo_cred_seteuid_exit = mac_do_seteuid_exit,
    
    // ... and similar triplets for all other syscalls
};
```

**The Result: Comprehensive Coverage**  
This implementation allows credential transitions from any standard POSIX credential-changing function, not only `setcred(2)`, to be authorized by `mac_do(4)`. With `mac_do(4)` control, applications can now be deployed without setuid bits and still execute the required **privilege** **escalations**. All syscalls use the same rule syntax; for example, `uid=1000>uid=0` will allow transitions to root through `setuid(2)`, `seteuid(2)`, or any other suitable syscall. **Per-thread data isolation**, appropriate cleanup on exceptions, and uniform rule application across all code paths are among the safety guarantees that the infrastructure upholds, just like the original `setcred(2)` implementation.

## mdo(1) improvements

`mdo(1)` is the companion userland program to `mac_do(4)` used for changing credentials by requesting `mac_do(4)` using the novel `setcred(2)` system call. Initially, it could only change to a target user.

I had two objectives for this program:

1. Extend `mdo(1)` to allow setting the **primary** **group** and secondary groups list, allowing **explicit** **UID** and **GID** overrides, and enabling users to have **fine**\-**grain** control over the credentials.
    
2. Allowing users to print the target part of `mac_do(4)` rules corresponding to the requested transition.
    

### Objective 1: From Simple User Switching to Fine-Grained Credential Control

The original `mdo(1)` utility was quite basic; it could switch to a different user and either keep the current groups or replace them entirely with the target user's groups. The interface was minimal:

```c
while ((ch = getopt(argc, argv, "u:i")) != -1) {
    switch (ch) {
    case 'u':
        username = optarg;
        break;
    case 'i':
        uidonly = true;  // Keep current groups
        break;
    }
}
```

The enhanced version now supports a comprehensive set of options for precise credential control:

```bash
# Basic usage remains the same
mdo -u alice id

# But now you can do much more:
mdo -u bob -g wheel -G staff,operator /bin/sh
mdo -u alice -s @,+wheel,+operator /usr/bin/id  
mdo --ruid 1002 --svgid 1003 --egid 1004 /bin/id
```

Managing usernames and numeric IDs correctly was one of the initial difficulties. This logic was disjointed and a little brittle in the original code. I developed specific functions for parsing and checking stuff.

```c
static uid_t parse_user_pwd(const char *s, struct passwd **pwd);
static uid_t parse_user(const char *s);
static gid_t parse_group(const char *s);
static gid_t *realloc_groups(gid_t *array, size_t new_count);
static int id_cmp(const size_t *id_a, const size_t *id_b);
static size_t remove_duplicates(gid_t *array, size_t count);
static size_t remove_groups_from_array(gid_t *array, size_t count, const gid_t *remove_list, size_t remove_count);
```

Then in the `main()` function, I started handling each option, starting with the user-related options, which were more straightforward than the group handling. I performed the relevant checks related to the absence of `-i`, `-g` or any explicit GID override, then moved on to setting the explicit UID options too. The most complex part was implementing the group management features. The new tool needed to support three different ways of specifying groups:

**1\. Exact Group Lists (**`-G`**):** Specify precisely which supplementary groups to have:

```bash
mdo -u bob -G wheel,staff,operator /bin/sh
```

**2\. Group Modifications (**`-s`**):** Add/remove groups with a powerful syntax:

```bash
mdo -u alice -s @,+wheel,+operator,-staff /usr/bin/id
```

**3\. Smart Defaults:** Inherit from the user database or the current process as appropriate.

The group modification syntax was particularly interesting to implement:

```c
while ((tok = strsep(&p, ",")) != NULL) {
    if (*tok == '\0')
        continue;
    if (tok[0] == '@')  {
        if (i > 0)
            errx(EXIT_FAILURE, "'@' must be the first token in -s option");
        supp_groups_reset = true;  // Start with empty group list
    } else if (tok[0] == '+' || tok[0] == '-') {
        bool is_add = tok[0] == '+';
        const char *gstr = tok + 1;
        gid = parse_group(gstr);
        if (is_add) {
            supp_groups_add = realloc_groups(supp_groups_add, add_count + 1);
            supp_groups_add[add_count++] = gid;
        } else {
            groups_supp_del = realloc_groups(groups_supp_del, rem_count + 1);
            groups_supp_del[rem_count++] = gid;
        }
    }
}
```

**Managing Dynamic Arrays and Memory:**  
Working with supplementary groups meant dealing with dynamic arrays that could grow and shrink. I needed utility functions for array manipulation:

```c
static size_t
remove_duplicates(gid_t *array, size_t count)
{
    size_t j = 0;
    if (count <= 1)
        return (count);
    qsort(array, count, sizeof(gid_t), gid_cmp);
    // Remove consecutive duplicates
    for (size_t i = 1; i < count; i++) {
        if (array[i] != array[j]) {
            array[++j] = array[i];
        }
    }
    return (j + 1);
}
```

Much more intricate **decision** **trees** had to be handled by the updated version. For instance, explicitly specifying users is incompatible with the -k flag (keep current user):

```c
if (start_from_current_user && (username_provided || ruid_str != NULL ||
    svuid_str != NULL || euid_str != NULL))
    errx(EXIT_FAILURE, "-k incompatible with -u or specifying all users");
```

And when we don't have a passwd entry but need to set groups, we must have explicit group specifications:

```c
if (!start_from_current_groups && pw == NULL && primary_group == NULL &&
    (rgid_str == NULL || svgid_str == NULL || egid_str == NULL))
    errx(EXIT_FAILURE,
        "must specify primary groups or a user with an entry in the password database");
```

### Objective 2: Implementing --print-rule.

The only thing I had to do here was add this at the end:

```c
if (print_rule) {
	fprintf(stdout, "uid=%u,gid=%u,", wcred.sc_uid, wcred.sc_gid);
	if (setcred_flags & SETCREDF_SUPP_GROUPS && wcred.sc_supp_groups_nb > 0) {
		fprintf(stdout, "+gid=");
		for (size_t i = 0; i < wcred.sc_supp_groups_nb; i++) {
			if (i > 0)
				fprintf(stdout, ",");
			fprintf(stdout, "%u", wcred.sc_supp_groups[i]);
		}
	}
	fprintf(stdout, "\n");
	exit(0);
}
```

## Conclusion

I tried to fit as many technicalities as I could include in a small article, but truth be told, the actual project was way more involved than what I could show here. Those unending kernel crashes, VM restarts, hair-tearing moments while debugging. With every problem I tried to solve, it felt like how little I knew, and I had to learn everything on the go. With this, My Google Summer of Code Journey comes to an end. Hope you liked it!

# TL;DR

**Go and read it.**
