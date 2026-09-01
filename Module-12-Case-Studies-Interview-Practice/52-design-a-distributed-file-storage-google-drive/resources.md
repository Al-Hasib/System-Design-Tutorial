# Further Reading & References

- [Content-addressable storage](https://en.wikipedia.org/wiki/Content-addressable_storage) -- Wikipedia overview of storing/retrieving data by content hash, the basis for chunk deduplication in this design.
- [Rsync](https://en.wikipedia.org/wiki/Rsync) -- Wikipedia entry on the rsync algorithm and delta encoding, the classic technique behind efficient delta sync of changed file regions.
- [Raft (algorithm)](https://en.wikipedia.org/wiki/Raft_(algorithm)) -- Wikipedia overview of the Raft consensus algorithm used for strongly consistent metadata replication.
- [The Raft Consensus Algorithm](https://raft.github.io/) -- official Raft site, with the paper, visualization, and reference implementations.
- [Amazon S3 Documentation](https://docs.aws.amazon.com/s3/) -- official docs for a widely used object storage service, representative of the block/chunk storage layer described in this design.
- [Apache Hadoop Distributed File System (HDFS) Architecture](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HdfsDesign.html) -- official docs describing a distributed, replicated, block-based file system with a similar metadata/block-storage split.
