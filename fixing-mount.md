[PR 4775](https://github.com/chef/chef/pull/4775) reverted a well-intentioned attempt to make the `mount` resource be options-aware. During the investigation some fundamental issues arose:

- Mounted filesystems are not convergent objects. They generally can't modify their attributes without unmounting them, which is essentially destroy+re-create. That is sometimes okay, but not in this case.

- Making `mount` options-aware produced cases where a user thought they were just changing the filesystem's options, but under the hood, that forced a dangerous unmount+remount.

- As something of a complication, the resource combines the concepts of a mounted filesystem and an entry in /etc/fstab, edits /etc/fstab directly, and uses it for some of its mounted filesystem shenanigans.

***
## Going Forward

### Making `mount` Safe

1. Strict: Add an `unmount_ok` boolean attribute, which defaults to `false`. If this is false, refuse to unmount a filesystem unless the `:umount` (not `:remount`) action is explicitly specified. The CCR then fails-safe, by never implicitly unmounting a filesystem.

1. Less Strict: Instead of adding an attribute, only allow unmounting if the action is `:umount` or `:remount`.

### Options Behavior
- We default the `:options` attributes to the string `"defaults"`, which many systems will translate to a reasonable set of defaults. _(This is the current behavior.)_

- If the user sets `:options` to something else, they must specify _all_ the mount options: the `mount` resource will not merge their options with the defaults. _(This is the current behavior.)_

- If the mount options have changed (according to the existing `#mount_options_unchanged?` method), but the user has not met the safety criteria above, the CCR fails as the resource is unable to bring the filesystem into the target state.