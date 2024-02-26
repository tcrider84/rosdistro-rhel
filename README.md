# NOTE: This repo is to be used alongside the COPR packages found at:
https://copr.fedorainfracloud.org/coprs/tcrider/autosd-ros1/


# Steps for automation:
( please note this is still a non-product approach as it contains packages from epel and ported from fedora to centos in copr):
```
$ sudo dnf install -y bash bzip2 coreutils cpio diffutils findutils gawk glibc-minimal-langpack grep gzip info patch redhat-rpm-config rpm-build sed shadow-utils tar unzip util-linux which xz glibc-devel lm_sensors yum

$ sudo dnf config-manager --set-enabled crb
$ sudo dnf install epel-release epel-next-release

$ sudo dnf install -y gcc-c++ python3-rosdep python3-rosinstall_generator python3-vcstool python3-pyopengl python3-gnupg

$ export ROS_OS_OVERRIDE=rhel
$ sudo rosdep init

$ wget https://raw.githubusercontent.com/ros/rosdistro/master/rosdep/sources.list.d/20-default.list
$ sed -i 's|yaml https://raw.githubusercontent.com/ros/rosdistro/master/rosdep/base.yaml|#yaml https://raw.githubusercontent.com/ros/rosdistro/master/rosdep/base.yaml|g' 20-default.list
$ sed -i 's|yaml https://raw.githubusercontent.com/ros/rosdistro/master/rosdep/python.yaml|#yaml https://raw.githubusercontent.com/ros/rosdistro/master/rosdep/python.yaml|g' 20-default.list
$ echo 'yaml https://raw.githubusercontent.com/tcrider84/rosdistro-rhel/main/base.yaml' >> 20-default.list
$ echo 'yaml https://raw.githubusercontent.com/tcrider84/rosdistro-rhel/main/python.yaml' >> 20-default.list
$ sudo mv 20-default.list /etc/ros/rosdep/sources.list.d/

$ rosdep update

$ mkdir ~/ros_catkin_ws
$ cd ~/ros_catkin_ws

$ rosinstall_generator desktop --rosdistro noetic --deps --tar > noetic-desktop.rosinstall
$ mkdir ./src
$ vcs import --input noetic-desktop.rosinstall ./src

$ sudo wget https://copr.fedorainfracloud.org/coprs/tcrider/autosd-ros1/repo/centos-stream+epel-next-9/tcrider-autosd-ros1-centos-stream+epel-next-9.repo -O /etc/yum.repos.d/copr-autosd-ros1.repo
$ sudo dnf config-manager --save --setopt="copr*autosd*ros1*.priority=50"

$ sudo dnf install -y sbcl ogre-devel python3-qt5 python3-qt5-webkit log4cxx-devel python3-mock python3-nose --refresh

$ rosdep install --from-paths ./src --ignore-packages-from-source --rosdistro noetic -y

$ cd src/rosconsole
$ wget https://github.com/ros/rosconsole/pull/58.patch
$ patch -Np1 < 58.patch
$ cd ../../

$ sed -i 's|Boost REQUIRED python|Boost REQUIRED python3.9|g' ~/ros_catkin_ws/src/vision_opencv/cv_bridge/CMakeLists.txt

$ ./src/catkin/bin/catkin_make_isolated --install -DCMAKE_BUILD_TYPE=Release
```

# Verify finished build files exist:
```
$ ls install_isolated/
bin  env.sh  etc  include  lib  local_setup.bash  local_setup.sh  local_setup.zsh  setup.bash  setup.sh  _setup_util.py  setup.zsh  share
```

# Notes regarding the above steps and various bugs/issues resolved:
-------------------------
1. In `/etc/ros/rosdep/sources.list.d/20-default.list` we modified the base.yaml and python.yaml files that ROS uses. Within our custom versions we added keys for rhel for the packages it needs.

Custom versions:  
yaml https://raw.githubusercontent.com/tcrider84/rosdistro-rhel/main/base.yaml  
yaml https://raw.githubusercontent.com/tcrider84/rosdistro-rhel/main/python.yaml  

Original error(s) reported:
```
ERROR: the following packages/stacks could not have their rosdep keys resolved
to system dependencies:

rqt_pose_view: No definition of [python3-opengl] for OS [rhel]
diagnostic_common_diagnostics: No definition of [lm-sensors] for OS [rhel]
webkit_dependency: No definition of [python3-qt5-bindings-webkit] for OS [rhel]
roslisp: [sbcl] defined as "not available" for OS version [*]
rosbag: No definition of [python3-gnupg] for OS [rhel]
rviz: No definition of [libogre] for OS [rhel]
gl_dependency: No definition of [python3-qt5-bindings-gl] for OS [rhel]
```

2. We use `export ROS_OS_OVERRIDE=rhel` because ROS does not detect our OS version. Recommend adding this in /etc/profile or ~/.bashrc for permanent use then relogin for it to take effect. Otherwise you have to set the export every time you run rosdep update.

Original error(s) reported:
```
ERROR: Rosdep experienced an error: Could not detect OS, tried ['zorin', 'windows', 'nixos', 'clearlinux', 'ubuntu', 'slackware', 'rocky', 'rhel', 'raspbian', 'qnx', 'pop', 'osx', 'sailfishos', 'tizen', 'conda', 'oracle', 'opensuse', 'opensuse', 'opensuse', 'opensuse', 'opensuse', 'openembedded', 'neon', 'mx', 'mint', 'linaro', 'gentoo', 'funtoo', 'freebsd', 'fedora', 'elementary', 'elementary', 'debian', 'cygwin', 'euleros', 'centos', 'manjaro', 'buildroot', 'arch', 'amazon', 'alpine', 'almalinux']
```

3. The packages in the copr repository are dependencies ROS requires that are shipped in Fedora but not in Centos. They have been rebuilt on centos for ROS.

Original error(s) reported:
```
ERROR: Rosdep experienced an error: Could not detect OS, tried ['zorin', 'windows', 'nixos', 'clearlinux', 'ubuntu', 'slackware', 'rocky', 'rhel', 'raspbian', 'qnx', 'pop', 'osx', 'sailfishos', 'tizen', 'conda', 'oracle', 'opensuse', 'opensuse', 'opensuse', 'opensuse', 'opensuse', 'openembedded', 'neon', 'mx', 'mint', 'linaro', 'gentoo', 'funtoo', 'freebsd', 'fedora', 'elementary', 'elementary', 'debian', 'cygwin', 'euleros', 'centos', 'manjaro', 'buildroot', 'arch', 'amazon', 'alpine', 'almalinux']
```

```
No match for argument: log4cxx-devel
No match for argument: python3-mock
No match for argument: python3-nose
```

4. rosconsole needs a pending upstream patch in order to work with log4cxx versions newer than 0.11 :

    Bug noted here:  
    https://github.com/ros/rosconsole/issues/50  
    
    Fix:__
    https://github.com/ros/rosconsole/pull/58  

5. Even though centos ships python3-qt5, there are two bugs with it with ROS which require us to rebuild.

The first is that ROS requires python3-qt5-webkit, which is only built on fedora, but not centos:
```
webkit_dependency: No definition of [python3-qt5-bindings-webkit] for OS [rhel]
```
```
$ rosdep resolve python3-qt5-bindings-webkit
                fedora:
                - python3-qt5-webkit
```

https://src.fedoraproject.org/rpms/python-qt5/blob/rawhide/f/python-qt5.spec#_153
```
%if 0%{?fedora}
%global webkit 1
%endif
```

So we have to remove the if/endif check in the spec sheet for enabling webkit.

The second is that ROS requires SIP 4 support in python:

Original error(s) reported:
```
Running SIP generator for qt_gui_cpp_sip Python bindings...
sip: Deprecation warning: qt_gui_cpp.sip:1: %Module version number should be specified using the 'version' argument
sip: /usr/lib64/python3.9/site-packages/PyQt5/bindings/QtCore/QtCoremod.sip:23: syntax error
```

Bug noted here:  
https://github.com/ros-noetic-arch/ros-noetic-python-qt-binding/issues/7

Arch Linux used to provide a patch to re-enable SIP support:  
https://github.com/archlinux/svntogit-packages/commit/9f73b4aabafa235823c529e3a37799ce678b776d#diff-442c918e6d9646aa3a16151aeba4ba9fe85d88e54e8e952da466e89ca11ba8b7

So I've rebased and added the patch to the python-qt5 build in COPR. This is also why we give the COPR repo higher priority:

```
$ sudo dnf config-manager --save --setopt="copr*autosd*ros1*.priority=100"
```
Because we override the python3-qt5 packages from centos, giving the repo higher priority ensures they won't be overridden again when a package update from centos occurs.

6. CMake in ROS does not correctly detect Boost python in centos, we have to modify it:
```
sed -i 's|Boost REQUIRED python|Boost REQUIRED python3.9|g' ~/ros_catkin_ws/src/vision_opencv/cv_bridge/CMakeLists.txt
```

Original error(s) reported:
```
CMake Error at /usr/share/cmake/Modules/FindPackageHandleStandardArgs.cmake:230 (message):
  Could NOT find Boost (missing: python) (found version "1.75.0")
```
-------------------------

And that's it!


TODO:

- Reproduce once more on a clean build to verify in case anything is missing
- Create a list of packages coming from epel.
