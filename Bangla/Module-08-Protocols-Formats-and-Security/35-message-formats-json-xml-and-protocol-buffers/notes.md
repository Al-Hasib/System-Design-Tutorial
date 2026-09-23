# অধ্যয়ন নোট: Message Formats — JSON, XML & Protocol Buffers

## সংজ্ঞাসমূহ (Definitions)

- **Serialization:** transmission বা storage-এর জন্য in-memory ডেটাকে একটি byte sequence-এ রূপান্তর করা।
- **Deserialization:** একটি byte sequence-কে আবার ব্যবহারযোগ্য in-memory ডেটাতে রূপান্তর করা।
- **Schema:** একটি message-এর structure-এর একটি formal সংজ্ঞা — field name, type, এবং (protobuf-এর ক্ষেত্রে) numeric tag — যাতে প্রেরক এবং প্রাপক উভয়েই সম্মত।
- **Schema evolution:** সেই নিয়ম/পদ্ধতিগুলো যা একটি message format-কে সময়ের সাথে পরিবর্তন হতে দেয় (field যোগ/অপসারণ) communicate করা service-গুলোর পুরনো এবং নতুন version-এর মধ্যে compatibility না ভেঙে।

## Format তুলনা

| দিক | JSON | XML | Protocol Buffers |
|---|---|---|---|
| Format type | Text | Text | Binary |
| Human-readable | হ্যাঁ | হ্যাঁ | না (schema + tooling দরকার) |
| Schema | Optional (JSON Schema একটি add-on) | Native, শক্তিশালী (XSD) | আবশ্যক (`.proto` file), enforced |
| Payload size | মাঝারি (field name পুনরাবৃত্ত) | বড় (verbose tag) | ছোট (wire-এ name নয়, field number) |
| Parse/serialize speed | মাঝারি | ধীর (tag parsing overhead) | দ্রুত (binary, কোনো tokenizing নেই) |
| Type safety | দুর্বল (int/float পার্থক্য নেই, enforced type নেই) | মাঝারি (XSD-এর মাধ্যমে) | শক্তিশালী (generated typed code) |
| Binary data support | দুর্বল (base64 দরকার, +~৩৩% size) | দুর্বল (একই সমস্যা) | Native |
| সাধারণ ব্যবহার | Public REST API, config, log | Legacy/enterprise integration, SOAP | Internal service-to-service (gRPC), high-throughput system |

## Schema Evolution নিয়মাবলী (Protocol Buffers)

- প্রতিটি field-এর একটি স্থায়ী numeric tag থাকে — field name নয়, এটিই আসলে wire-এ encode হয়।
- **নিরাপদ:** একটি নতুন tag number সহ নতুন field যোগ করা (পুরনো code অচেনা tag উপেক্ষা করে)।
- **অনিরাপদ:** একটি বিদ্যমান field-এর tag পুনরায় ব্যবহার/পুনঃসংখ্যায়ন করা (তাহলে পুরনো এবং নতুন code একটি নির্দিষ্ট tag-এর অর্থ নিয়ে দ্বিমত হবে)।
- **সুপারিশকৃত:** একটি field সরানোর সময়, এর tag `reserved` হিসেবে চিহ্নিত করা যাতে এটি পরে ভুলবশত অন্য field-এ বরাদ্দ না হয়।
- JSON/XML এটিকে আরও শিথিলভাবে সামলায় — একটি পুরনো consumer সাধারণত অচেনা field উপেক্ষা করে এবং অনুপস্থিত field-কে null/absent হিসেবে গণ্য করে, উভয় ক্ষেত্রেই কোনো enforced contract ছাড়া।

## সিদ্ধান্ত গ্রহণের নির্দেশিকা (Decision Guide)

| যদি আপনার অগ্রাধিকার হয়... | তাহলে বেছে নিন... |
|---|---|
| Human debuggability / বিস্তৃত tooling / public API | JSON |
| শক্তিশালী native schema validation, enterprise/legacy interop | XML |
| ন্যূনতম bandwidth, দ্রুততম (de)serialization, internal RPC | Protocol Buffers (সাধারণত gRPC-এর মাধ্যমে) |
| আপনার নিজের service-গুলোর মধ্যে streaming বা অত্যন্ত উচ্চ call volume | Protocol Buffers / gRPC |

## গুরুত্বপূর্ণ সংখ্যা/তথ্য

- Binary data-কে base64-encode করা (JSON/XML text-এ embed করার জন্য প্রয়োজনীয়) এর size-এ প্রায় ৩৩% যোগ করে।
- Protobuf payload সাধারণত একই ডেটার সমতুল্য JSON-এর চেয়ে ৩-১০ গুণ ছোট, যা field type এবং পুনরাবৃত্তির উপর নির্ভর করে।
- Protocol Buffers Google-এ অভ্যন্তরীণভাবে তৈরি হয়েছিল এবং ২০০৮ সালে open-source করা হয়েছিল।

## সারসংক্ষেপ

- প্রতিটি message format হলো human-readability/tooling (JSON, XML) এবং wire efficiency/type-safety (Protocol Buffers)-এর মধ্যে একটি trade-off।
- JSON হলো public এবং browser-facing API-এর জন্য বাস্তবসম্মত default; XML মূলত legacy/enterprise integration-এ টিকে আছে; Protocol Buffers high-throughput internal service communication-এ প্রাধান্য বিস্তার করে, বিশেষ করে gRPC-এর সাথে জুটিবদ্ধ হয়ে।
- Schema evolution — বিদ্যমান consumer-দের না ভেঙে কীভাবে field যোগ/অপসারণ করবেন — service-গুলো স্বাধীনভাবে deploy এবং version করা হলে এটি একটি প্রথম-শ্রেণির design উদ্বেগ।
</content>
