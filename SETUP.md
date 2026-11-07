# Hadoop single-node setup, state, changes, and remaining work

This file records the state the machine was in, what was verified, what was
changed and why, and how to finish HW #2.

Last verified: 2026-11-07, cluster running, WordCount smoke test passed on YARN.

---

## 1. What was already installed (unchanged by me)

| Item | Value |
| --- | --- |
| Hadoop | 3.5.0, distro package (AUR/pacman) |
| Binaries | `/usr/bin/{hadoop,hdfs,yarn,mapred}`, `/usr/sbin/start-*.sh`, `stop-*.sh` |
| Libraries | `/usr/share/hadoop/{common,hdfs,mapreduce,yarn,tools}` |
| Shell library | `/usr/libexec/hadoop-{config,functions}.sh` |
| Stock config | `/etc/hadoop` (root-owned, `644`) |
| Daemon logs | `/var/log/hadoop` (owned by `malik`, writable) |
| Java in `PATH` | OpenJDK **27** (Hadoop 3.5 does not support this) |
| `JAVA_HOME` for Hadoop | `/usr/lib/jvm/java-17-openjdk`, set in `hadoop-env.sh`, correct |
| SSH | `sshd` active, `ssh localhost` works passwordless, required by `start-dfs.sh` |
| `workers` | `localhost` |

Environment exports from `/etc/profile.d/hadoop.sh`:
`HADOOP_CONF_DIR=/etc/hadoop`, `HADOOP_LOG_DIR=/var/log/hadoop`,
`HADOOP_COMMON_LIB_NATIVE_DIR=/usr/lib`.

There is no tarball install, so **`HADOOP_HOME` is unset**. The equivalent of
`$HADOOP_HOME` on this machine is `/usr`. This matters later.

Previous attempt evidence: `jps` was empty, nothing listened on `:9000`, no
NameNode had ever been formatted, and `/var/log/hadoop` holds DataNode logs
from 2026-09-20. So Hadoop had been started once and cleanly stopped/shut down,
but the filesystem was never formatted.

## 2. Checks that passed as-is

- `hadoop version` → 3.5.0, runs on Java 17.
- `ssh localhost` → `SSH_OK`, no password. `start-dfs.sh` needs this.
- `hadoop conftest` → all XML well-formed.
- `hadoop classpath` resolves; `javac` 17 compiles `src/WordCount.java`
  including its `org.apache.log4j.Logger` import (provided by Hadoop's
  `reload4j`).
- Disk: 273 GB free on `/home`. RAM: 15 GB total.

## 3. Defects found (this is why 2006.csv would fail)

### D1. All Hadoop state was in RAM (`/tmp` is tmpfs)

`hadoop.tmp.dir` was unset, so it defaulted to `/tmp/hadoop-malik`.
On this machine `/tmp` is a **7.7 GB tmpfs**, i.e. RAM.

That single fact puts the NameNode image, the **DataNode block files**, and the
YARN container/shuffle directories in memory. A 672 MB dataset plus shuffle
spill does not fit comfortably, fails with `No space left on device` /
`DiskChecker$DiskErrorException`, and vanishes on reboot. This is the classic
"small dataset works, big dataset fails" cause.

### D2. `mapred-site.xml` was empty → Hadoop was in *local* mode

`mapreduce.framework.name` defaults to `local` when unset. `hadoop jar` would
run inside the client JVM and never contact YARN: no ResourceManager job, no
containers, and a 1 GB client heap doing all the work on 672 MB.

### D3. `yarn-site.xml` was empty → no shuffle service

Without `yarn.nodemanager.aux-services=mapreduce_shuffle`, every MapReduce job
on YARN dies in the reduce phase with
`Shuffle failed with too many fetch failures`.

### D4. YARN virtual-memory check (latent)

Default `yarn.nodemanager.vmem-check-enabled=true` with a 2.1x- physical ratio.
The smoke-test job already showed a map task reserving **2.8 GB virtual** inside
a 1 GB container. It only survived because it was disabled, with defaults it
would have been killed with `running beyond virtual memory limits`.

### D5. `HADOOP_MAPRED_HOME=$HADOOP_HOME` is wrong for a distro install

The reference manual writes `HADOOP_MAPRED_HOME=$HADOOP_HOME`. Here
`HADOOP_HOME` is unset, so that string expands to nothing and the MRAppMaster
container cannot find its jars
(`Could not find or load main class ...v2.app.MRAppMaster`). `/usr` is the
correct value on this machine.

## 4. What I changed, and how

`/etc/hadoop` is root-owned and interactive `sudo` was unavailable in this
session, so instead of editing it the config was moved to a **user-owned copy
inside the project** and `HADOOP_CONF_DIR` was pointed at it. No root needed,
config is version-controlled with the assignment, and a package upgrade cannot
overwrite it.

### 4.1 New config directory

```
cp -a /etc/hadoop/. /home/malik/Projects/hadoop/conf/
```

`conf/hadoop-env.sh` is kept as-is, it already pins
`JAVA_HOME=/usr/lib/jvm/java-17-openjdk`, which is essential because the
default `java` on this box is 27.

### 4.2 `conf/core-site.xml`

| Property | Value | Why |
| --- | --- | --- |
| `fs.defaultFS` | `hdfs://localhost:9000` | kept from the original |
| `hadoop.tmp.dir` | `/home/malik/hadoop_tmp` | **D1**, off tmpfs, onto the 273 GB disk |

### 4.3 `conf/hdfs-site.xml`

| Property | Value | Why |
| --- | --- | --- |
| `dfs.replication` | `1` | kept; one DataNode |
| `dfs.namenode.name.dir` | `file:///home/malik/hadoop_tmp/dfs/name` | explicit, so nobody has to infer it from `hadoop.tmp.dir` |
| `dfs.datanode.data.dir` | `file:///home/malik/hadoop_tmp/dfs/data` | same |
| `dfs.client.socket-timeout` | `600000` | `hdfs dfs -cat` of a large `part-r-00000` otherwise times out at 60 s |

Block size left at the default 128 MB → a ~672 MB file becomes ~6 splits, ~6
mappers.

### 4.4 `conf/mapred-site.xml` (was empty)

| Property | Value | Why |
| --- | --- | --- |
| `mapreduce.framework.name` | `yarn` | **D2** |
| `yarn.app.mapreduce.am.env` | `HADOOP_MAPRED_HOME=/usr` | **D5** |
| `mapreduce.map.env` | `HADOOP_MAPRED_HOME=/usr` | **D5** |
| `mapreduce.reduce.env` | `HADOOP_MAPRED_HOME=/usr` | **D5** |

### 4.5 `conf/yarn-site.xml` (was empty)

| Property | Value | Why |
| --- | --- | --- |
| `yarn.nodemanager.aux-services` | `mapreduce_shuffle` | **D3** |
| `...aux-services.mapreduce.shuffle.class` | `org.apache.hadoop.mapred.ShuffleHandler` | **D3** |
| `yarn.nodemanager.local-dirs` | `/home/malik/hadoop_tmp/yarn/local` | **D1**, container jars and shuffle data |
| `yarn.nodemanager.log-dirs` | `/home/malik/hadoop_tmp/yarn/log` | **D1** |
| `yarn.nodemanager.resource.memory-mb` | `6144` | 15 GB box with ~8 GB already used by the desktop; 8 GB default is unrealistic |
| `yarn.scheduler.maximum-allocation-mb` | `6144` | must be ≥ any single request, else the job is rejected outright |
| `yarn.nodemanager.vmem-check-enabled` | `false` | **D4**, physical memory accounting stays on |

### 4.6 `~/.bashrc`

Added above the interactive-shell guard, so every shell that reads the file
gets it:

```
export HADOOP_CONF_DIR="$HOME/Projects/hadoop/conf"
```

This only changes the **client** side. `start-dfs.sh` / `start-yarn.sh` pass
`--config "${HADOOP_CONF_DIR}"` explicitly to the daemons they launch over ssh,
so the daemons get the same directory even though non-interactive ssh shells do
not read `~/.bashrc`.

### 4.7 State directories

```
mkdir -p /home/malik/hadoop_tmp          # populated by the format below
hdfs namenode -format -nonInteractive    # ONCE, do not repeat, it erases HDFS
```

### 4.8 Build

```
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk
export PATH=$JAVA_HOME/bin:$PATH
javac -classpath "$(hadoop classpath)" -d build/wc_class src/WordCount.java
jar -cvf build/wordcount.jar -C build/wc_class .
```

Compiled with **javac 17** on purpose: `javac` in `PATH` is 27, and its class
files will not load in Hadoop's Java 17 JVM.

## 5. Verification

```
$ jps
131105 SecondaryNameNode
130610 NameNode
131993 ResourceManager
132382 NodeManager
130846 DataNode
```

Resolved values:

```
fs.defaultFS                       = hdfs://localhost:9000
hadoop.tmp.dir                     = /home/malik/hadoop_tmp
dfs.replication                    = 1
dfs.blocksize                      = 134217728
mapreduce.framework.name           = yarn
yarn.nodemanager.aux-services      = mapreduce_shuffle
yarn.nodemanager.vmem-check-enabled = false
yarn.nodemanager.resource.memory-mb = 6144
```

Smoke test (3 tiny files, `data/smoke/`):

```
$ hadoop jar build/wordcount.jar WordCount \
    /user/malik/wordcount/input /user/malik/wordcount/output_smoke
...
Shuffled Maps =3   Failed Shuffles=0   Bytes Read=87   Bytes Written=84
```

```
$ yarn application -list -appStates ALL
application_1794011896662_0001  wordcount  MAPREDUCE  malik  root.default  FINISHED  SUCCEEDED  100%
```

That last line is the important one: `FINISHED / SUCCEEDED` means it really ran
as a YARN application with containers, not in local mode.

Output was correct:

```
.	1
Hadoop	3
is	3
as	2
yellow	2
a	1  an	1  be	1  can	1  elephant	1  fellow	1  Oh	1  what	1
```

## 6. If you would rather have this in `/etc/hadoop` instead

The project-local copy is what is active now. To move it system-wide, replace
the four files with the ones in `conf/`:

```
sudo cp ~/Projects/hadoop/conf/{core-site,hdfs-site,mapred-site,yarn-site}.xml /etc/hadoop/
```

and then remove the `HADOOP_CONF_DIR` export from `~/.bashrc` (leaving
`/etc/profile.d/hadoop.sh` in charge). Everything else is identical, and
`hadoop_tmp` stays where it is. Do **not** run both variants against the same
`hadoop_tmp` at the same time, same data, so it works, but it is easy to get
confused about which config produced what.

## 7. Daily commands

Every new terminal already has `HADOOP_CONF_DIR` set. Sanity check:

```
hdfs getconf -confKey hadoop.tmp.dir     # must print /home/malik/hadoop_tmp
```

Start / stop / status:

```
start-dfs.sh && start-yarn.sh
jps
stop-yarn.sh && stop-dfs.sh
```

Web UI: NameNode <http://localhost:9870>, ResourceManager <http://localhost:8088>.

Daemon startup logs: `/var/log/hadoop/hadoop-malik-<daemon>-omarchy.log`.

---

## 8. Remaining work, the assignment itself

Deadline: **2026-11-19 (Thu) before 15:00**.

### 8.1 Combine and extract the dataset

```
cd ~/Projects/hadoop/data
cat 2006.tar.gzaa 2006.tar.gzab > 2006.tar.gz
tar -xzf 2006.tar.gz
ls -lh
```

`2006.tar.gzaa` starts with the gzip magic `1f 8b` and `2006.tar.gzab` starts
with `b7 42`, i.e. the second file is a raw continuation of the first. `cat` in
that order is correct. Expect ~672 MB after extraction.

### 8.2 Put the input into HDFS

Copies the file into the DataNode directory on real disk, so make sure the
cluster is running first.

```
hdfs dfs -mkdir -p /user/malik/wordcount/input_2006
hdfs dfs -put data/2006.csv /user/malik/wordcount/input_2006/
hdfs dfs -ls -h /user/malik/wordcount/input_2006
```

### 8.3 Run the job, capturing progress and results in one file

The output directory must not already exist, or the job fails immediately with
`Output directory already exists`.

```
cd ~/Projects/hadoop
hadoop jar build/wordcount.jar WordCount \
    /user/malik/wordcount/input_2006 \
    /user/malik/wordcount/output_2006 \
    2>&1 | tee output_AhsanMalikAlFarisi_50261896.txt

hdfs dfs -cat /user/malik/wordcount/output_2006/* \
    >> output_AhsanMalikAlFarisi_50261896.txt
```

The first command writes the progress log, the second appends the final counts.
`tee` rather than a plain redirect keeps the log visible while it runs, which is
what the assignment example does (`2>&1 | tee`).

**Confirm the filename against your student ID before submitting**, I used the
name from a PDF already in `~/Downloads`.

Optional, if the single reducer is too slow on 672 MB (default is 1 reducer):

```
-D mapreduce.job.reduces=4
```

then `hdfs dfs -getmerge` the output instead of `-cat *`.

### 8.4 Convert to Windows line endings

The assignment requires `todos` (from `tofrodos`), which **does not exist in the
Arch repositories**, it is an Ubuntu/Debian package. `dos2unix` is in `extra`
and provides the same CRLF conversion:

```
sudo pacman -S dos2unix
unix2dos output_AhsanMalikAlFarisi_50261896.txt
file output_AhsanMalikAlFarisi_50261896.txt
```

`file` must report `with CRLF line terminators`. That is the −5 point check.

### 8.5 Online submission

1. Attach `output_AhsanMalikAlFarisi_50261896.txt`.
2. In the text field, answer both A and B:
   - **A**: the errors that occurred on 2006.csv. If the run is clean, write
     "No issues", the output file is the required evidence.
   - **B**: how you fixed them. Sections 3 and 4 of this file are exactly that
     material: state was on a tmpfs (`/tmp`), the install was in local mode, the
     shuffle service was missing, and YARN's virtual-memory check killed
     containers. Each has the property, the old value, the new value.

## 9. Gotchas

- **Never re-run `hdfs namenode -format`** on this directory. It erases HDFS.
- `WARN util.NativeCodeLoader: Unable to load native-hadoop library` appears on
  every command. Harmless, the native libs are simply not built for this
  platform; Hadoop falls back to pure Java.
- `HADOOP_HOME` is unset by design here; do not copy `$HADOOP_HOME` paths from
  the reference manual. The install prefix is `/usr`.
- If `start-dfs.sh` prints `Connection refused` for ssh, the daemons are being
  launched but `ssh localhost` broke, test it on its own.
- If a job is rejected before starting, check
  `yarn.scheduler.maximum-allocation-mb` against the requested memory.
- `/home/malik/hadoop_tmp` is the whole cluster state. Deleting it means
  reformatting and re-uploading.
