# A dnf rollback helper

### Synopsis:

To roll back to a transaction:

`>dnf-rollback --tid=123 --exec`

where 123 is the dnf history transaction ID.

### Installation:

This script uses python3
Note: Only tested on AlmaLinux 9 - may work on other RHEL-based distros.

```
>wget https://github.com/arnon-weinberg/dnf-rollback/raw/master/dnf-rollback
>chmod +x dnf-rollback
```

### Details:

I find that dnf history rollback often just does not work. The [docs](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_software_with_the_dnf_tool/assembly_handling-package-management-history_managing-software-with-the-dnf-tool#reverting-dnf-transactions_assembly_handling-package-management-history) say:

> Downgrading RHEL system packages to an older version by using the dnf history undo and dnf history rollback command is not supported. This concerns especially the selinux, selinux-policy-*, kernel, and glibc packages, and dependencies of glibc such as gcc. Therefore, downgrading a system to a minor version (for example, from RHEL 9.1 to RHEL 9.0) is not recommended because it might leave the system in an incorrect state.

But there are many other problems: Error messages are incomprehensible, packages are missed, and the operation is atomic, so often the entire rollback fails because of one package.

If dnf history rollback works for you, then great. Otherwise, use this script to help debug problems, or as an alternative to dnf history rollback. The following is a tutorial on how to roll back transactions.

To find a transaction ID to roll back to, use:

`>dnf history`

Additional details of a transaction are provided by:

`>dnf history info 123`

dnf also supports transaction ID `last` and `last-#`; for example:

`>dnf history info last-4`

is the transaction 4th from last.

I think of a rollback as the process of moving the system from one list of package-versions to another. To generate the current (move-from) list, it's possible to:

`>rpm -qa >pkgs.last`

But for the purposes of diff, it's better to use:

`>dnf-rollback --tid=last >pkgs.last`

To generate the target (move-to) list, use:

`>dnf-rollback --tid=123 >pkgs.123`

or use `--tid=last-4` or whatever. Now you can:

`>diff pkgs.last pkgs.123`

to see what changes are needed. If ready to proceed, then the next step is to generate the commands needed to perform the move:

`>dnf-rollback --pkgs=pkgs.123 --comm >comm.123`

Notice that the list can just be loaded from a file to save time, instead of processing the transaction history again. This provides an opportunity to make changes to the list, if you want to roll back to a custom state.

The commands are for dnf shell. To execute them, use:

`>sudo dnf shell comm.123`

Note that dnf shell still prompts you to commit, so this is safe to run. The command file provides another opportunity to customize results, work around problems, or try different solutions.

If no customization is needed, then it's possible to do this:

`>(dnf-rollback --pkgs=pkgs.123 --comm && cat) | sudo dnf shell`

Though this will leave you in the shell, so you'll need to exit manually. Alternatively, you can just:

`>dnf-rollback --pkgs=pkgs.123 --exec`

This should complete the process, though again, nothing is committed until you confirm. I often run both:

`>sudo dnf history rollback 123`

and

`>dnf-rollback --tid=123 --exec`

side-by-side to compare results before committing. I am always happier with dnf-rollback of course, and unlike dnf history rollback, things that are not working can be fixed.

To pass additional arguments to dnf shell, use:

`>dnf-rollback --tid=123 --exec -- --assumeno --setopt=install_weak_deps=False`

arguments after `--` are passed to dnf shell.

A note about the vault:

Rolling back to previous system versions is often stymied by missing or disabled repos. AlmaLinux moves older packages to the vault repos, but does not provide a repo file for the vaults. To generate a repo file for the vault, I use:

`>generate-almalinux-vault-repo | sudo tee /etc/yum.repos.d/almalinux-vault.repo >/dev/null`

Note that these repos are disabled by default, as they should be. To see the full list of repos - both enabled and disabled - use:

`>dnf repolist --all`

To get an idea of what vaults are likely to be needed, try:

`>dnf list installed | tail -n +2 | awk '{print $3}' | sort | uniq -c`

Then enable them as required:

`>sudo dnf config-manager --set-enabled baseos-9.4`

etc. Remember to disable them when done the rollback.
