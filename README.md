# swarm #

A robust and fast clustering method for amplicon-based studies.

The purpose of **swarm** is to provide a novel clustering algorithm to handle large sets of amplicons. Traditional clustering algorithms results are strongly input-order dependent, and rely on an arbitrary **global** clustering threshold. **swarm** results are resilient to input-order changes and rely on a small **local** linking threshold *d*, the maximum number of differences between two amplicons. **swarm** forms stable high-resolution clusters, with a high yield of biological information.

## Quick start ##

**swarm** most simple usage is (with default parameters, see user manual for details):

```
./swarm amplicons.fasta
```

### Warning ###

**swarm** only runs on CPUs with SSE4.1 instructions. These instructions were introduced by Intel in November 2007 for servers and January 2008 for desktop and portable CPUs. It has been supported by AMD CPUs since October 2011. **swarm** should be able to run on any Intel or AMD CPU released since.

## Install ##

Get the source code and a **swarm** binary from [GitHub](https://github.com/torognes/swarm "swarm public repository") using the [ZIP button](https://github.com/torognes/swarm/archive/master.zip "swarm zipped folder") or git:

```
git clone https://github.com/torognes/swarm.git
cd swarm/
```

Use the command `make` to compile **swarm** from scratch.

If you have administrator privileges, you can make **swarm** accessible for all users. Simply copy the binary to `/usr/bin/`. The man page can be installed this way:

```
gzip -c swarm.1 > swarm.1.gz
mv swarm.1.gz /usr/share/man/man1/
```

Once installed, the man page can be accessed with the command `man swarm`.

## Prepare amplicon fasta files ##

To facilitate the use of **swarm**, we provide examples of shell commands that can be use to format and check the input fasta file (warning, this may not be suitable for very large files). The amplicon clipping step (adaptor and primer removal) and the filtering step are not discussed here.

### Linearization ###

Amplicons written on two lines are easier to manipulate: one line for the fasta header, one line for the sequence (tested with GNU Awk 4.0.1).

```
awk 'NR==1 {print ; next} {printf /^>/ ? "\n"$0"\n" : $1} END {print}' amplicons.fasta > amplicons_linearized.fasta
```

### Dereplication ###

To speed up the clustering process, strictly identical amplicons should be merged. This step is not mandatory, but it is an important time saver, especially for highly redundant high-throughput sequencing results.

```
grep -v "^>" amplicons_linearized.fasta | sort -d | uniq -c |
while read abundance sequence ; do
    hash=$(echo ${sequence} | sha1sum)
    hash=${hash:0:40}
    printf ">%s_%d_%s\n" "${hash}" "${abundance}" "${sequence}"
done | sort -t "_" -k2,2nr | sed -e 's/\_/\n/2' > amplicons_linearized_dereplicated.fasta
```

The dereplicated amplicons receive a meaningful unique name, and are sorted by decreasing number of copies. The use of a hashing function also provides an easy way to compare sets of amplicons. If two amplicons from two different sets have the same hash, it means that they have identical sequences.

### Search for duplicated amplicon names ###

**swarm** does not check if your amplicons have unique names. To avoid ambiguous results, test your fasta file with this bash command (it should print nothing, unless you have duplicated amplicon names):

```
grep -o -E "^>\S+" amplicons.fasta | tr -d ">" | sort -d | uniq -d
```

### Launch swarm ###

If you want **swarm** to partition your dataset with the finest resolution (a local number of differences *d* = 1) on a quadricore CPU:

```
./swarm -d 1 -t 4 amplicons.fasta > amplicons.swarms
```

See the user manual (man page and PDF) for details on **swarm**'s options and parameters.

## Parse swarm results ##

To facilitate the use of **swarm**, we provide examples of shell commands that can be use to parse **swarm**'s output. We assume that the amplicon fasta file was prepared as describe above (linearization and dereplication).

### Statistics ###

For each swarm, print the number of unique amplicons, the number of copies, the name of the seed and its abundance, and the number of singletons (amplicons with an abundance of 1). When using input data sorted by decreasing abundance, the seed is the most abundant amplicon in the swarm (tested with GNU Awk 4.0.1).

```
awk 'BEGIN {OFS="\t"} {sum=0 ; singletons=0 ; seed=$1 ; sub("_", "\t", seed) ; for (i=1 ; i<=NF ; i++) {split($i, amplicon, "_") ; sum+=amplicon[2] ; if (amplicon[2] = 1) singletons++}} {print NF, sum, seed, singletons}' amplicons.swarms
```

### Get the seed sequence for each swarm ###

It is frequent for subsequent analyses to keep only one representative amplicon per OTU (usually the seed) to reduce the computational burden. That operation is easily done with **swarm** results.

```
SEEDS=$(mktemp)
cut -d " " -f 1 amplicons.swarms | sed -e 's/^/>/' > "${SEEDS}"
grep -A 1 -F -f "${SEEDS}" amplicons.fasta | sed -e '/^--$/d' > amplicons_seeds.fasta
rm "${SEEDS}"
```

### Get fasta sequences for all amplicons in a swarm ###

For each swarm, get the fasta sequences for all amplicons. Warning, this loop can generate a very large number of files. To limit the number of files, a test can be added to exclude swarms with less than *n* elements.

```
AMPLICONS=$(mktemp)
mkdir swarms_fasta
while read swarm ; do
    tr " " "\n" <<< "${swarm}" | sed -e 's/^/>/' > "${AMPLICONS}"
    seed=$(head -n 1 "${AMPLICONS}")
    grep -A 1 -F -f "${AMPLICONS}" amplicons.fasta | sed -e '/^--$/d' > "./swarms_fasta/${seed}.fasta"
done < amplicons.swarms
rm "${AMPLICONS}"
```

### Convert swarm results into the uclust output format ###

Some pipelines use the [uclust output format](http://www.drive5.com/uclust/uclust_userguide_1_1_579.html#_Toc257997686 "page describing the uclust output format") as input for subsequent analyses. The following Awk command efficiently converts **swarm**'s output into the tab-separated text uclust output format, using a fasta file and a swarm file as input:

```
awk -v fasta_file="amplicons.fasta" 'BEGIN {RS="\n*>"; FS="\n" ; while (getline < fasta_file) {lengths[$1] = length($2)} ; close(fasta_file) ; RS = "\n" ; FS = " " ; OFS = "\t" ; NA = "*"} {seed = $1 ; number = NR -1 ; {print "C", number, NF, NA, NA, NA, NA, NA, seed, NA "\n" "S", number, lengths[$1], NA, NA, NA, NA, NA, seed, NA} ; {for (i=2 ; i <= NF ; i++) {print "H", number, lengths[$i], NA, "+", 0, 0, NA, $i, seed}}}' amplicons.swarms
```

The command is a bit complex but very efficient. Tests on a laptop show that it converts a swarm file containing 10-million amplicons names in less than two minutes. Warning, as of today the columns %Id and Alignment are filled with "*". In future versions of **swarm**, we will try to provide these data.