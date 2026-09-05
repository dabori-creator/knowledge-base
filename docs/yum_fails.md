# yum fails with rpmlib dependency issue

## **Environment**

- Red Hat Enterprise Linux 8

## **Issue**

- *yum* failed with below error:

```
Error: transaction check vs depsolve: rpmlib(CaretInVersions) <= 4.15.0-1 is needed by python39- pip-20.2.4-3.module+el8.4.0+9822+20bf1249.noarch
rpmlib(CaretInVersions) <= 4.15.0-1 is needed by python39-pip-wheel- 20.2.4-7.module+el8.6.0+13003+6bb2c488.noarch
```

## **Resolution**

- Execute below command to update dnf package

```
yum update "*dnf*" libsolv
```

- Confirm if still seeing this issue using *yum* :

```
yum update python39\*
```

- If that succeeds, proceed to package installation/update

```
yum install <package_name>
```

## **Root Cause**

- *yum* failed to install/update packages due to installed *dnf libsolv* versions are too old.

```
$ grep dnf installed-rpms
dnf-4.0.9.2-5.el8.noarch                                    Thu    Aug    6 
11:24:09 2020
dnf-data-4.0.9.2-5.el8.noarch                                Thu    Aug    6
11:23:27 2020
dnf-plugins-core-4.0.2.2-3.el8.noarch                        Thu    Aug    6
11:24:09 2020
dnf-utils-4.0.2.2-3.el8.noarch                                Thu    Aug    6
11:24:10 2020
libdnf-0.22.5-4.el8.x86_64                                    Thu    Aug    6
11:24:09 2020
python3-dnf-4.0.9.2-5.el8.noarch                            Thu    Aug    6
11:24:09 2020
python3-dnf-plugins-core-4.0.2.2-3.el8.noarch               Thu    Aug    6
11:24:09 2020
python3-libdnf-0.22.5-4.el8.x86_64                            Thu    Aug    6
11:24:09 2020


$ grep libsolv installed-rpms libsolv-0.6.35-6.el8.x86_64    Thu    Aug 6
11:23:43 2020            
```

## **Diagnostic Steps**

- verify the `*dnf*` libsolv packages install date.

```
# rpm -qa | grep dnf
# rpm -qa | grep libsolv
```
