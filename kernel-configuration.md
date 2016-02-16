1. List module status in kernel: `lsmod`
2. Module configuration: `/etc/modprobe.d`
3. List module files: `find /lib/modules/`uname -r` -type f -name "*.ko"`
4. List modules built into kernel: `/lib/modules/\`uname -r\`/modules.builtin`. See [this](http://askubuntu.com/questions/666880/loop-module-not-present-on-ubuntu-installation).
5. [Difference between loadable and builtin module](http://stackoverflow.com/questions/22929065/linux-loadable-modules-and-built-in-modules)
