# আরও পড়ার জন্য ও Reference

- [Content-addressable storage](https://en.wikipedia.org/wiki/Content-addressable_storage) -- content hash দিয়ে data store/retrieve করার Wikipedia overview, যা এই design-এ chunk deduplication-এর ভিত্তি।
- [Rsync](https://en.wikipedia.org/wiki/Rsync) -- rsync algorithm এবং delta encoding-এর Wikipedia entry, পরিবর্তিত file অংশের efficient delta sync-এর পেছনের classic technique।
- [Raft (algorithm)](https://en.wikipedia.org/wiki/Raft_(algorithm)) -- strongly consistent metadata replication-এর জন্য ব্যবহৃত Raft consensus algorithm-এর Wikipedia overview।
- [The Raft Consensus Algorithm](https://raft.github.io/) -- Raft-এর official site, যেখানে paper, visualization, এবং reference implementation আছে।
- [Amazon S3 Documentation](https://docs.aws.amazon.com/s3/) -- ব্যাপকভাবে ব্যবহৃত একটি object storage service-এর official docs, যা এই design-এ বর্ণিত block/chunk storage layer-এর প্রতিনিধিত্বমূলক।
- [Apache Hadoop Distributed File System (HDFS) Architecture](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HdfsDesign.html) -- একই ধরনের metadata/block-storage বিভাজন সহ একটি distributed, replicated, block-based file system বর্ণনা করা official docs।
