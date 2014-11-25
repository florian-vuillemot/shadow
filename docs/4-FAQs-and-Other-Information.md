1. **Shadow is running at 100% CPU. Is that normal?**  
Yes. In single-thread mode, Shadow runs at 100% CPU because its continuously processing simulation events as fast as possible. All other things constant, an experiment will finish quicker with a faster CPU. Due to node dependencies, thread CPU utilization will be less than 100% in multi-thread mode.

2. **Is Shadow multi-threaded?**  
Yes. Shadow can run with _N_ worker threads by specifying `-w N` or `--workers=N` on the command line. Note that virtual nodes depend on network packets that can potentially arrive from other virtual nodes. Therefore, each worker can only advance according to the propagation delay to avoid dependency violations.

3. **Is it possible to achieve deterministic experiments, so that every time I run Shadow with the same configuration file, I get the same results?**  
Yes. You need to use the "--cpu-threshold=-1" flag when running Shadow to disable the CPU model, as it introduces non-determinism into the experiment in exchange for more realistic CPU behaviors. (See also: `shadow --help-all`)

4. **Can I use Shadow/Scallion with my custom Tor modifications?**  
Yes. You'll need to build Shadow with the `--tor-prefix` option set to the path of your Tor source directory. Then, every time you make Tor modifications, you need to rebuild and reinstall Shadow and Scallion, again using the `--tor-prefix` option.

# Other Notes

## Documentation

Documentation may be generated by running doxygen:
```bash
doxygen Doxyfile
```

Then check the `doc/` directory. Documentation is currently sparse.

## Debugging

When debugging, it will be helpful to use the Shadow option `--cpu-threshold=-1`. It disable the automatic virtual CPU delay measurement feature. This feature may introduce non-deterministic behaviors, even when running the exact same experiment twice, by the re-ordering of events that occurs due to how the kernel schedules the physical CPU of the experiment machine. Disabling the feature with the above option will ensure a deterministic experiment, making debugging easier. (See [this issue](https://github.com/shadow/shadow/issues/45))

Build Shadow with debugging symbols by using the `-g` flag. See the help menu with `python setup.py build --help`.

You will need to set `LD_PRELOAD` when running in gdb. Its value should contain a colon separated list of every 'preload' library (generally prefixed with `libshadow-preload` and installed to `~/.shadow/lib`):
```
gdb shadow-bin
> set environment LD_PRELOAD=/home/rob/.shadow/lib/libshadow-preload.so
> set args --echo
> run
```

If this doesn't work and you just see "exited with code 1", instead set
`LD_PRELOAD` in gdb as follows:
```
> set exec-wrapper env LD_PRELOAD=/home/rob/.shadow/lib/libshadow-preload.so
```

The following example shows how to run a scallion experiment in gdb:

```
gdb shadow-bin
> set env EVENT_NOSELECT=1
> set env EVENT_NOPOLL=1
> set env EVENT_NOKQUEUE=1
> set env EVENT_NODEVPOLL=1
> set env EVENT_NOEVPORT=1
> set env EVENT_NOWIN32=1
> set env OPENSSL_ia32cap=~0x200000200000000
> set env LD_PRELOAD=/home/rob/.shadow/lib/libshadow-preload.so:/home/rob/.shadow/lib/libshadow-preload-scallion.so
> set args /home/rob/.shadow/share/topology.xml /home/rob/test/small-m1.xlarge/hosts.xml
> run
```

If you want to be able to run Shadow through valgrind and the application you 
are running in Shadow uses OpenSSL (i.e. the Scallion plug-in), you should configure OpenSSL with the 
additional option: `-DPURIFY`. This fixes OpenSSL so it doesn't break valgrind.
You may also want to ensure that debugging symbols are included in the GLib
that Shadow links to, and any library used by the plug-in. This can be achieved
with the compiler flag `-g` when manually building a local version of GLib.

__NOTE__: Currently, symbols from plug-ins are not accessible via GDB until [this issue](https://github.com/shadow/shadow/issues/101) gets fixed. See [this thread](http://mailman.cs.umn.edu/archives/shadow-dev/2013-September/000066.html) for more information.

## Tagging new versions

The following commands can be used to tag a new version of Shadow, after which an
archive and signature can be uploaded to github's download section, and linked
from the Shadow website.

```bash
git tag -s v1.5.0
git push --tags
git checkout release
git merge -Xtheirs v1.5.0
git rm {DELETED-FILE-NAMES}
git archive --prefix=shadow-v1.5.0/ --format=tar v1.5.0 | gzip > shadow-v1.5.0.tar.gz
gpg -a -b shadow-v1.5.0.tar.gz
gpg --verify shadow-v1.5.0.tar.gz.asc
```

Note that `git merge -Xtheirs v1.5.0` will assume all conflicts can be fixed by taking the remote tag/branch that you are merging into the current branch. This will keep our releases following master.