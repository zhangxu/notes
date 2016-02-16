1. List loadable modules: `lsmod`
2. Module configuration: `/etc/modprobe.d`
3. List module files: `find /lib/modules/`uname -r` -type f -name "*.ko"`
4. List modules built into kernel: `/lib/modules/\`uname -r\`/modules.builtin`. See [this](http://askubuntu.com/questions/666880/loop-module-not-present-on-ubuntu-installation).
