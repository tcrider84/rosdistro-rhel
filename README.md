# NOTE: This repo is to be used alongside the COPR packages found at:
https://copr.fedorainfracloud.org/coprs/tcrider/autosd-ros1/


# Steps for automation:
( please note this is still a non-product approach as it contains packages from epel and ported from fedora to centos in copr):
```

# Install base dependencies for building ros:

$ sudo dnf install -y bash bzip2 coreutils cpio diffutils findutils gawk glibc-minimal-langpack grep gzip info patch redhat-rpm-config rpm-build sed shadow-utils tar unzip util-linux which xz glibc-devel lm_sensors yum

# Enable EPEL and CRB:

$ sudo dnf config-manager --set-enabled crb
$ sudo dnf install epel-release epel-next-release

# Install EPEL dependencies for building ros:

$ sudo dnf install -y gcc-c++ python3-rosdep python3-rosinstall_generator python3-vcstool python3-pyopengl python3-gnupg

# Force rhel as ROS_OS_OVERRIDE to allow rosdep init to work:

$ export ROS_OS_OVERRIDE=rhel
$ sudo rosdep init

# Fix missing ros dependencies:

$ wget https://raw.githubusercontent.com/ros/rosdistro/master/rosdep/sources.list.d/20-default.list
$ sed -i 's|yaml https://raw.githubusercontent.com/ros/rosdistro/master/rosdep/base.yaml|#yaml https://raw.githubusercontent.com/ros/rosdistro/master/rosdep/base.yaml|g' 20-default.list
$ sed -i 's|yaml https://raw.githubusercontent.com/ros/rosdistro/master/rosdep/python.yaml|#yaml https://raw.githubusercontent.com/ros/rosdistro/master/rosdep/python.yaml|g' 20-default.list
$ echo 'yaml https://raw.githubusercontent.com/tcrider84/rosdistro-rhel/main/base.yaml' >> 20-default.list
$ echo 'yaml https://raw.githubusercontent.com/tcrider84/rosdistro-rhel/main/python.yaml' >> 20-default.list
$ sudo mv 20-default.list /etc/ros/rosdep/sources.list.d/

# Add more missing dependencies ported from fedora to copr repo:

$ sudo wget https://copr.fedorainfracloud.org/coprs/tcrider/autosd-ros1/repo/centos-stream+epel-next-9/tcrider-autosd-ros1-centos-stream+epel-next-9.repo -O /etc/yum.repos.d/copr-autosd-ros1.repo
$ sudo dnf config-manager --save --setopt="copr*autosd*ros1*.priority=50"

$ sudo dnf update python3-qt5 sip --refresh
$ sudo dnf install -y sbcl ogre-devel python3-qt5-webkit log4cxx-devel python3-nose --refresh

$ rosdep update

# Create building directory:

$ mkdir ~/ros_catkin_ws
$ cd ~/ros_catkin_ws

# Prepare ros sources:

$ rosinstall_generator desktop --rosdistro noetic --deps --tar > noetic-desktop.rosinstall
$ mkdir ./src
$ vcs import --input noetic-desktop.rosinstall ./src

$ rosdep install --from-paths ./src --ignore-packages-from-source --rosdistro noetic -y

# allow newer versions of log4cxx to work within sources:

$ cd src/rosconsole
$ wget https://github.com/ros/rosconsole/pull/58.patch
$ patch -Np1 < 58.patch
$ cd ../../

# fix python3-boost detection on RHEL within sources:

$ sed -i 's|Boost REQUIRED python|Boost REQUIRED python3.9|g' ~/ros_catkin_ws/src/vision_opencv/cv_bridge/CMakeLists.txt

# fix python3-mock within sources -- python3-mock is deprecated, replaced by unittest: https://fedoraproject.org/wiki/Changes/DeprecatePythonMock#How_to_migrate_to_unittest.mock

$ sed -i 's/import mock/from unittest import mock/g' ./src/ros_comm/rosgraph/test/test_xmlrpc.py
$ sed -i 's/from mock import /from unittest.mock import /g' ./src/rqt_tf_tree/test/dotcode_tf_test.py
$ sed -i 's/from mock import /from unittest.mock import /g' ./src/rqt_dep/test/dotcode_pack_test.py
$ sed -i 's/from mock import /from unittest.mock import /g' ./src/catkin/test/unit_tests/test_environment_cache.py
$ sed -i 's/from mock import /from unittest.mock import /g' ./src/catkin/test/unit_tests/test_find_in_workspace.py
$ sed -i 's/from mock import /from unittest.mock import /g' ./src/catkin/test/unit_tests/test_parse_package_xml.py

# build

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

The second is that python-qt5's ros module for qt gui requires py_ssize_t_clean:  

Original error(s) reported:  
```
[ 84%] Built target qt_gui_cpp
[ 89%] Running SIP generator for qt_gui_cpp_sip Python bindings...
sip: Deprecation warning: qt_gui_cpp.sip:1: %Module version number should be specified using the 'version' argument
sip: /usr/lib64/python3.9/site-packages/PyQt5/bindings/QtCore/QtCoremod.sip:23: syntax error
Traceback (most recent call last):
  File "/home/tcrider/ros_catkin_ws/install_isolated/share/python_qt_binding/cmake/sip_configure.py", line 122, in <module>
    subprocess.check_call(cmd)
  File "/usr/lib64/python3.9/subprocess.py", line 373, in check_call
    raise CalledProcessError(retcode, cmd)
subprocess.CalledProcessError: Command '['/usr/bin/sip', '-c', '/home/tcrider/ros_catkin_ws/build_isolated/qt_gui_cpp/sip/qt_gui_cpp_sip', '-b', '/home/tcrider/ros_catkin_ws/build_isolated/qt_gui_cpp/sip/qt_gui_cpp_sip/pyqtscripting.sbf', '-I', '/usr/lib64/python3.9/site-packages/PyQt5/bindings', '-w', '-n', 'PyQt5.sip', '-t', 'Qt_5_15_0', '-t', 'WS_X11', 'qt_gui_cpp.sip']' returned non-zero exit status 1.
make[2]: *** [src/qt_gui_cpp_sip/CMakeFiles/libqt_gui_cpp_sip.dir/build.make:103: sip/qt_gui_cpp_sip/Makefile] Error 1
make[1]: *** [CMakeFiles/Makefile2:337: src/qt_gui_cpp_sip/CMakeFiles/libqt_gui_cpp_sip.dir/all] Error 2
```

The bug is noted here:  

Bug noted here:  
https://github.com/ros-noetic-arch/ros-noetic-python-qt-binding/issues/7  

Arch Linux used to provide a patch to re-enable SIP support:  
https://github.com/archlinux/svntogit-packages/commit/9f73b4aabafa235823c529e3a37799ce678b776d#diff-442c918e6d9646aa3a16151aeba4ba9fe85d88e54e8e952da466e89ca11ba8b7  

However this is going backwards. Instead of doing that the proper fix is to backport upstream sip changes that include py_ssize_t_clean when calling the module.  

Someone else has also already done that with the following patch:  

https://raw.githubusercontent.com/robwoolley/meta-openembedded/ad1494b75f6ece452bdaec88c32772a32cd9186b/meta-oe/recipes-devtools/sip/sip3/added-the-py_ssize_t_clean-argument-to-the-module-directive.patch  

I've rebased the patch and rebuilt sip with it in our copr repo.  

Since we are recompiling both sip and python-qt5, we need to give our repo higher priority so it knows to install sip and python-qt5 from our repo instead of the default autosd repo:  
```
$ sudo dnf config-manager --save --setopt="copr*autosd*ros1*.priority=100"
```

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

