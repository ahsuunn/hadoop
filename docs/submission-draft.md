# HW2 online submission draft

Paste the text of item A and item B below into the e-class submission field.
The only attachment is the output file, no separate document is required.

Verified against the run of 2026-11-07 17:14:37, `job_1794038568243_0001`.

---

## Item A

No issues. The job ran on the full 2006.csv (640.9 MB, 672,084,480 bytes read by
HDFS) and completed successfully on the first attempt in 3 minutes 55 seconds:
5 input splits, 5 map tasks plus 1 speculative duplicate that Hadoop killed, 1
reduce task, and 12,905 distinct words in the output. The job appears in the
ResourceManager history as application_1794038568243_0001 with state
FINISHED / SUCCEEDED. Before running the dataset, the cluster was checked against
the failure modes that typically appear on a single-node installation and three
configuration problems were found and corrected. They are described in item B,
since they are the single-node system limitations this task refers to.

## Item B

Three settings were corrected during setup, before 2006.csv was processed. No
error occurred while processing the dataset itself.

First, HDFS storage was on memory. `hadoop.tmp.dir` was unset and therefore
defaulted to `/tmp/hadoop-<user>`, and on this machine `/tmp` is a 7.7 GB tmpfs,
so the NameNode image, DataNode block files and YARN container directories would
all have been held in RAM. I moved `hadoop.tmp.dir`, `dfs.namenode.name.dir`,
`dfs.datanode.data.dir`, `yarn.nodemanager.local-dirs` and
`yarn.nodemanager.log-dirs` to `/home/malik/hadoop_tmp` on disk.

Second, `mapred-site.xml` and `yarn-site.xml` were empty, which leaves MapReduce
in local mode and gives YARN no shuffle service. I set
`mapreduce.framework.name=yarn`, `yarn.nodemanager.aux-services=mapreduce_shuffle`
with `org.apache.hadoop.mapred.ShuffleHandler`, and the three `*.env` properties
to `HADOOP_MAPRED_HOME=/usr`, because `$HADOOP_HOME` is not set on a package
installation.

Third, the YARN virtual memory check was at its default. The map container's
virtual memory peaked at 2.63 GiB while its physical allocation was 1 GiB, and
YARN permits virtual memory only up to 2.1 times the allocation, that is 2.1 GiB,
so the container would have been killed for exceeding its virtual memory limit.
I set `yarn.nodemanager.vmem-check-enabled=false` and kept physical memory
accounting enabled.

The job was then run with:

    hadoop jar build/wordcount.jar WordCount \
      /user/malik/wordcount/input_2006 \
      /user/malik/wordcount/output_2006 \
      2>&1 | tee output_AhsanMalikAlFarisi_50261896.txt

and the counts were appended with:

    hdfs dfs -cat /user/malik/wordcount/output_2006/* \
      >> output_AhsanMalikAlFarisi_50261896.txt

---

## Evidence, if a grader asks for it

| Item | Value |
| --- | --- |
| Application ID | `application_1794038568243_0001` |
| Final state | FINISHED / SUCCEEDED |
| Start / end | 17:14:37 / 17:18:32 (3 min 55 s) |
| Input | 2006.csv, 640.9 MB, 672,084,480 bytes read |
| Splits / tasks | 5 splits, 6 map tasks launched (1 speculative kill), 1 reduce |
| Map input records | 7,141,923 |
| Reduce output records | 12,905 |
| Output written | 132,079 bytes |
| Output file | 13,021 lines, 12,905 counts + 116 log lines, CRLF line terminators |

An independent second run on the same input produced byte-identical output
(`md5 453d36a3f55211db454262e21c2ca341`), so the result is reproducible.

## Before submitting

1. Confirm the filename. The assignment wants
   `output_<Name>_<StudentID>.txt`; the file is named
   `output_AhsanMalikAlFarisi_50261896.txt` on the assumption that the name and
   ID are those already printed on other course files. Check the format the
   course uses.
2. Confirm the CRLF conversion. `file output_*.txt` must report
   `with CRLF line terminators`. `todos` from `tofrodos` does not exist in the
   Arch repositories; `unix2dos` from `dos2unix` does the same conversion. This
   is the -5 point check.
3. Wording of A and B. Item A already says "No issues" and item B opens by
   saying no error occurred during the run, so the two cannot be read as a
   contradiction. If you still prefer to submit item A on its own, replace
   the last sentence of A ("Before running the dataset...") with:
   "Before running the dataset the cluster was checked against the failure
   modes that typically appear on a single-node installation, and three
   configuration problems were found and corrected, so they produced no
   error in this run." and omit item B entirely. Be aware that this leaves
   the "main goal" paragraph of the directions answered only by the output
   file.

## Why the errors were never observed

Worth knowing in case you are asked in person. The three defects were found by
inspecting the configuration, not by watching a job fail, and they were fixed
before 2006.csv was ever run, so no error appeared on this machine:

- `hadoop.tmp.dir` resolving to `/tmp/hadoop-malik` and `/tmp` being a tmpfs were
  both measured. That the 640 MB input would have overflowed 7.7 GB was an
  inference, and not a certain one.
- The empty `mapred-site.xml` really does mean local mode, but a local-mode run
  would likely have completed, only slowly.
- The missing shuffle service only matters when the framework is `yarn`, so with
  the original configuration it was latent rather than active.
- The virtual memory figure is the one hard number: 2.82 GB peak in a 1 GB
  container, against a 2.15 GB default limit. That one would almost certainly
  have killed the tasks, but it was still not observed.

Do not claim an observed error in the submission.
