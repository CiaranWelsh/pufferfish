# What is pufferfish?

**short answer** : Pufferfish is a new time and memory-efficient data structure for indexing a compacted, colored de Bruijn graph (ccdBG).  

**long answer** : Though the de Bruijn Graph (dBG) has enjoyed tremendous popularity as an assembly and sequence comparison data structure, it has only relatively recently begun to see use as an index of the reference sequences (e.g. [deBGA](https://github.com/HongzheGuo/deBGA), [kallisto](https://github.com/pachterlab/kallisto)).  Particularly, these tools index the _compacted_ dBG (cdBG), in which all non-branching paths are collapsed into individual nodes and labeled with the string they spell out.  This data structure is particularly well-suited for representing repetitive reference sequences, since a single contig in the cdBG represents all occurences of the repeated sequence.  The original positions in the reference can be recovered with the help of an auxiliary "contig table" that maps each contig to the reference sequence, position, and orientation where it appears as a substring.  Moreoever, the cdBG can be built on multiple reference sequences (transcripts, chromosomes, genomes), where each reference is given a distinct color.  The resulting structure, which also encodes the relationships between the cdBGs of the underlying reference sequences is called the compacted, colored de Bruijn graph.

While existing hash-based indices based on the cdBG (and ccdBG) are very efficient for search, they typically occupy a large amount of space in memory (both during construction and even when built).  As a result, to make use of such data structures on large reference sequences (e.g., the human genome) or collections of reference sequences (e.g., in a metagenomic context), one typically requires a very large memory machine --- if the structures can be built at all.  Pufferfish implements a new and much more compact data structure for indexing the ccdBG.  While maintaining very efficient queries, this allows Pufferfish to index reference sequences while reducing the memory requirements considerably (by an order-of-magnitude or more).  This greatly reduces the memory burden for indexing reference sequences and makes it possible to build hash-based indexes of sequences of size that were not previously feasible.

**about pufferfish development:**
Currently, Pufferfish is the software implementing this efficient ccdBG index, and allowing point (i.e., k-mer) queries.  Pufferfish is under active development, but we want to be as open (and as useful to as many people) as possible early on. However, we are also in the process of building higher-level tools (e.g., read mappers and aligners) around this index, so stay tuned!

**branches:**
The **master** branch of pufferfish is _not_ necessarily stable, but it should, at any given time contain a working version of the index.  That is, breaking changes should not be pushed to master.  The **develop** branch of pufferfish is guaranteed to be neither stable nor working at any given point, but a best-faith effort will be made to not commit broken code to this branch.  For feature branches, all bets are off.

# Some details about pufferfish

Pufferfish supports (i.e., implements) two types of indices — dense and sparse.  Both of these make use of an efficient [minimum perfect hash implementation](https://github.com/rizkg/BBHash) to vastly reduce the memory requirements compared to other hashing-based dBG, colored-dBG and compacted-colored-dBG indices.  However, the sparse index saves even more space by only directly storing the position for certain k-mers (and inferring the positions dynamically for the rest).

### The basic (dense) pufferfish index

The basic (dense) pufferfish index can be visualized as in the image below:

![dense pufferfish index](https://github.com/COMBINE-lab/pufferfish/blob/master/doc/dense_index_diagram.jpg "dense pufferfish index")

The contigs of the compacted dBG are stored contiguously in a (2-bit encoded) vector, and boundary bit vector marks the end of each contig.  All of the `N` *valid* k-mers (i.e. k-mers not crossing contig boundaries) are indexed in a minimum perfect hash function.  A k-mer query (i.e., does this k-mer exist in the index, and if so, where does it occur in the reference?) is answered by consulting the MPHF h().  For a k-mer x, if `0 <= h(x) < N`, either `x != ContigSeqs[h(x) : h(x) + k]`, in which case `x` does not appear in the cDBG, or `PosTable[h(x)]` stores the position of `x` in `ContigSeqs`.  Efficient `rank` and `select` queries on the `ContigRank` vector can be used to determine the relative offset of `x` in the contig (say `c`) where it occurs, and to lookup all occurences of `c` in the reference sequence.  This information is encoded in the contig reference table (`CTab`), which records the reference sequence, offset and relative orientation of all occurences of `c` in the reference.

### The sparse pufferfish index

In addition to the tables maintained in the dense index, the sparse index also records 4 other pieces of information.  There are 3 extra bit-vectors (`isSampled`, `isCanonical`, and `extendDirection`) and one extra packed integer vector (`extendBases`).  The sparse index builds on the observation that any valid k-mer in the cDBG is:

1. a complex k-mer (it has an in or out degree > 1)
2. a terminal k-mer (it starts or ends some input reference sequence)
3. a distinct k-mer with precisely one successor / predecessor
  
Instead of storing the position for every k-mer (which requires lg( |ContigSeqs| ) bits per entry), the sparse index stores positions for only a sampled set of k-mers (where `IsSampled` is 1) — so that the `PosTable` becomes considerably smaller.  For k-mers whose positions are not stored, the `QueryExt` table records which nucleotides should be added to either the beginning (if `ExtRight` is 0) or end (if `ExtRight` is 1) of the query k-mer to move toward the nearest sampled position.  We also need to know if the variant of the k-mer that appears in the `ContigSeqs` table is canonical or not (hence the `IsCanon` vector).  In this scheme, successive queries and extensions move toward the nearest sampled k-mer until a position can be recovered, at which point the implied position of the query k-mer can be recovered and the rest of the query process proceeds as in the dense index.  By trading off the rate of sampling and the size of the extension that is stored (e.g., 1, 2, 3, ... nucleotides), one can trade-off between index size and query speed.  A sparse sampling and shorter extension require more queries to locate a k-mer, but result in a smaller index, while a denser sampling and/or longer extensions require fewer queries in the worst case, but more space.  If the extension size (`e`) is >= 1/2 the sampling rate (`s`), one can guarantee that at most one extra query need be performed when searching for a k-mer. The sparse index is currently in the `sampling` branch of pufferfish, and is in the process of being merged into the main code.


![sparse pufferfish extension](https://github.com/COMBINE-lab/pufferfish/blob/master/doc/sparse_index_diagram.jpg "dense pufferfish index")
