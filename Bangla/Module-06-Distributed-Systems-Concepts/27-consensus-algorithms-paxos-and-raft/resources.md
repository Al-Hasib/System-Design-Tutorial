# আরও পড়াশোনা ও তথ্যসূত্র (Further Reading & References)

## অফিসিয়াল ডকুমেন্টেশন (Official Docs)

- [The Raft Consensus Algorithm (raft.github.io)](https://raft.github.io/) — অফিসিয়াল Raft ওয়েবসাইট, যাতে leader election এবং log replication-এর সুপরিচিত interactive visualization অন্তর্ভুক্ত রয়েছে।
- [etcd Documentation — Raft in etcd](https://etcd.io/docs/latest/learning/) — etcd কীভাবে তার replication layer হিসেবে Raft consensus protocol implement করে, যা Kubernetes control-plane store হিসেবে ব্যবহৃত হয়।

## গবেষণাপত্র (Papers)

- Leslie Lamport, ["The Part-Time Parliament"](https://lamport.azurewebsites.net/pubs/lamport-paxos.pdf) (ACM Transactions on Computer Systems, 1998) — মূল Paxos paper, যা কাল্পনিক গ্রিক দ্বীপ Paxos-এর একটি আইনসভা সম্পর্কিত একটি রূপক (allegory) আকারে তৈরি।
- Leslie Lamport, ["Paxos Made Simple"](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf) (ACM SIGACT News, 2001) — Lamport-এর নিজের, আরও সরাসরি Paxos-এর পুনর্ব্যাখ্যা, যা লেখা হয়েছিল কারণ মূল paper-টি অনুসরণ করা খুব কঠিন ছিল।
- Diego Ongaro এবং John Ousterhout, ["In Search of an Understandable Consensus Algorithm (Raft)"](https://raft.github.io/raft.pdf) (USENIX ATC, 2014) — মূল Raft paper, যা স্পষ্টভাবে understandability-কে একটি first-class লক্ষ্য হিসেবে কেন্দ্র করে ডিজাইন করা।

## আরও পড়াশোনা (Further Reading)

- [Wikipedia — Paxos (computer science)](https://en.wikipedia.org/wiki/Paxos_(computer_science)) — Paxos-এর role, phase, এবং variant (Multi-Paxos, Fast Paxos) সম্পর্কে সহজবোধ্য একটি সংক্ষিপ্ত বিবরণ।
- [Wikipedia — Raft (algorithm)](https://en.wikipedia.org/wiki/Raft_(algorithm)) — Raft-এর leader election, log replication, এবং safety property সম্পর্কে সহজবোধ্য একটি সংক্ষিপ্ত বিবরণ।
