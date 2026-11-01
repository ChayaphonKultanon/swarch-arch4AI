# Tutorial 6 worksheet: serving patterns

**Name / ชื่อ: ชยพล กุลตานนท์**

**Student ID / รหัสนิสิต: 6631306021**

This is **individual work**: each student completes and submits their own copy, in their own fork.

*งานนี้เป็น**งานเดี่ยว** นิสิตแต่ละคนทำและส่งงานของตัวเอง โดยใช้ forked repository ของตัวเอง*

Copy result rows from `results/runs.csv` (or the load test's output) into the tables. Keep answers short: one to three sentences each. Answer in Thai or English.

ตอนที่ตอบ ให้นิสิตคัดลอกผลจาก `results/runs.csv` (หรือจาก output ของ load test) ลงในตาราง ตอบสั้นๆ ข้อละ 1–3 ประโยค จะตอบเป็นภาษาไทยหรือภาษาอังกฤษก็ได้

## Task 0: Baseline (copy the baseline results from `results/runs.csv` here / copy ผลลัพธ์ของ baseline จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|
| t0-large | 507 | 592 | 93.5% | 100% |
| t0-small | 11 | 38 | 72.5% | 0% |

**Q0.** What does the large model buy, and what does it cost?

*large model ให้อะไรเพิ่ม และต้องแลกด้วยอะไร*

> large model ให้ accuracy 93.5% เทียบกับ 72.5% ของ small model แต่ต้องแลกกับ latency ที่สูงกว่ามาก และ cost เพราะทุกรีวิวไปถึง large model ซึ่งทุก call มีค่าใช้จ่าย (QR4) และรับได้สูงสุดแค่ 20 calls ต่อวินาที (QR5)

## Task 1: Fallback prediction (copy the Task 1 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 1 จาก `results/runs.csv` มาวางที่นี่)

| Run | run time (s) | p50 ms | p95 ms | errors | pending | accuracy |
|---|---|---|---|---|---|---|
| t1-before | 26.7 s | 229 | 411 | 18 | 0 | 93.4% |
| t1-after | 9.6 s | 392 | 1010 | 0 | 28 | 93.0% |

**Q1.1** In `t1-before`, p50 can look *better* than the baseline. Why is p50 misleading here? Which column tells the real story?

*ใน `t1-before` ค่า p50 อาจดู "ดีกว่า" baseline ทำไม p50 จึงทำให้เข้าใจผิดในกรณีนี้ และ column ไหนบอกความจริง*

> p50 คิดจาก request ที่สำเร็จเท่านั้น 18 requests ที่ค้างจน client timeout (10 s) จึงไม่ถูกนับ และ request ที่ค้างยังกิน slot ของ load test client (ส่งพร้อมกันได้ 10) ไว้นาน 10 s ทำให้ request ที่แย่ง 4 slots ของ large model เหลือน้อยลงและคิวสั้นลงจนดูเร็วขึ้น (229 ms เทียบกับ 507 ms) column ที่บอกความจริงคือ `errors` = 18 ร่วมกับเวลาที่ run ใช้ (26.7 s เทียบกับประมาณ 10 s)

**Q1.2** The hang affects only 10% of large-model calls. Why is it worse for the system than 10% of reviews being slow? Think about what a waiting request holds on to in the review service and the gateway.

*การค้างเกิดกับ calls ไปยัง large model แค่ 10% ทำไมจึงส่งผลต่อระบบแย่กว่าการที่รีวิว 10% ช้าลงเฉยๆ ลองคิดว่า request ที่รออยู่ถือทรัพยากรอะไรไว้ใน review service และ gateway*

> request ที่ค้างถือ connection ไว้ทั้งใน gateway (timeout 60 s) และ review service (ไม่มี timeout) นานถึง 20 s แม้ client จะเลิกรอไปแล้ว ทำให้ request อื่นต้องรอทรัพยากรตามไปด้วย ผลคือ throughput ของทั้งระบบลดลง (run time เพิ่มจากประมาณ 10 s เป็น 26.7 s) ไม่ได้มีแค่ 10% ของรีวิวที่ช้า

**Q1.3** Pending reviews have no label. Suggest what should happen to them, and name the topology from the lecture that does it.

*รีวิวที่ pending ยังไม่มี label ควรจัดการกับรีวิวเหล่านี้อย่างไร และ topology ใดจาก lecture ที่ทำแบบนั้น*

> ควรเก็บรีวิว pending ไว้หรือส่งเข้าคิว แล้วให้ batch job มาติด label ด้วย large model ทีหลัง topology นี้คือ **batch processing** ใน lecture (inference in bulk, work queue)

**Q1.4** We chose a 1-second timeout. What goes wrong if it is 0.1 s? And if it is 10 s?

*เราเลือก timeout 1 วินาที จะเกิดปัญหาอะไรถ้าตั้งเป็น 0.1 วินาที และถ้าตั้งเป็น 10 วินาที*

> **0.1 s:** สั้นกว่าเวลาปกติของ large model (`t0-large` p50 = 507 ms) รีวิวเกือบทั้งหมดจึงจะกลายเป็น pending แม้โมเดลไม่ได้ค้าง **10 s:** เท่ากับ client timeout ผู้ใช้จึงยังเจอ error และ request ที่ค้างยังถือทรัพยากรไว้นานเหมือนใน Q1.2 ซึ่งผิด QR2 (รอได้ไม่เกิน 1 s)

## Task 2: Cascade (copy the Task 2 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 2 จาก `results/runs.csv` มาวางที่นี่)

| Run | threshold | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|---|
| t2-0.6 | 0.6 | 6 | 544 | 91.5% | 38% |
| t2-0.7 | 0.7 | 391 | 588 | 93.0% | 53% |
| t2-0.8 | 0.8 | 449 | 589 | 93.5% | 64% |
| t2-0.9 | 0.9 | 470 | 579 | 93.5% | 77% |

**Q2.1** Which threshold would you deploy for CampusEats, and why? Name the quality you give up.

*จะเลือก threshold ใดสำหรับ deploy ใน CampusEats เพราะอะไร และต้องยอมเสีย quality ใดไป*

> เลือก 0.6 เพราะใช้ large model น้อยที่สุด (38%) โดย accuracy 91.5% ยังผ่าน QR3 (≥ 90%) และ p95 544 ms ยังผ่าน QR1 (< 1 s) ซึ่งตรงกับ QR4 สิ่งที่ยอมเสียคือ accuracy (91.5% เทียบกับ 93.5% ของ `t0-large`) และเหลือ margin เหนือ QR3 แค่ 1.5 จุด

**Q2.2** Look at p50 and p95 for your threshold. Why does the cascade help p50 much more than p95?

*ดูค่า p50 และ p95 ของ threshold ที่เลือก ทำไม cascade จึงช่วย p50 ได้มากกว่า p95 มาก*

> ที่ 0.6 มี 62% ของ requests ที่ small model ตอบเอง (ไม่ถึง 1 ms) request ตรงกลางจึงเป็นของ small model และ p50 เหลือ 6 ms ส่วน 5% ที่ช้าที่สุดอยู่ในกลุ่ม 38% ที่ส่งต่อไป large model ซึ่งยังต้องจ่าย latency ของ large model เต็มๆ p95 จึงยังอยู่ที่ 544 ms

**Q2.3** The cascade trusts the small model's confidence. Give one situation where that trust is misplaced, and its effect on users.

*cascade เชื่อ confidence ของ small model ยกตัวอย่างหนึ่งสถานการณ์ที่ไม่ควรเชื่อ และผลที่เกิดกับผู้ใช้*

> รีวิวที่เป็นประโยคปฏิเสธ เช่น "not cold" ซึ่ง small model มักอ่านผิด ถ้า small model ตอบผิดแต่ confidence ถึง threshold รีวิวนั้นจะไม่ถูกส่งไป large model เลย (เห็นได้จาก accuracy ที่ลดจาก 93.5% ใน `t0-large` เหลือ 91.5% ใน `t2-0.6`) ผลคือเจ้าของร้านเห็นรีวิวชมเป็นลูกค้าไม่พอใจ และจำนวนรีวิวแยกตาม label ที่เห็นตาม FR3 ผิดไปจากจริง

## Task 3: Cached predictions (copy the Task 3 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 3 จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls | cache hits |
|---|---|---|---|---|---|
| t3-cache | 5 | 558 | 91.5% | 31% | 36% |

**Q3.1** Which reviews benefit from the cache, and which never will?

*รีวิวแบบไหนได้ประโยชน์จาก cache และแบบไหนไม่มีวันได้ประโยชน์*

> รีวิวที่ข้อความซ้ำกัน เช่น "Delicious" ได้ประโยชน์ เพราะตอบจาก cache ได้ตั้งแต่ครั้งที่ 2 (cache hits 36%, large calls ลดจาก 38% ใน `t2-0.6` เหลือ 31% โดย accuracy เท่าเดิมที่ 91.5%) ส่วนรีวิวยาวที่ไม่ซ้ำใคร หรือต่างกันแค่คำเดียวหรือเครื่องหมายวรรคตอนเดียว จะไม่มีวันได้ประโยชน์ เพราะ key ไม่ตรงกัน

**Q3.2** Next week you deploy a better large model. What must happen to the cache, and how would you design for it?

*สัปดาห์หน้าจะ deploy large model ตัวใหม่ที่ดีกว่าเดิม ต้องทำอะไรกับ cache และจะออกแบบรองรับเรื่องนี้อย่างไร*

> ต้อง invalidate cache เพราะคำตอบเดิมมาจากโมเดลเก่า ถ้าไม่ล้าง ข้อความที่เคยเห็นแล้วจะได้คำตอบเก่าต่อไป (ตอนนี้ key มีแค่ข้อความ ไม่มี version ของโมเดล) จึงควรออกแบบให้ใส่ model version ใน cache key เมื่อ deploy โมเดลใหม่ cache เก่าจะ miss เองโดยอัตโนมัติ ตามหลัก versioning models ใน lecture

## Debrief

**Q4.** Name one thing you learned from these measurements that the lecture did not cover.

*ยกตัวอย่างหนึ่งสิ่งที่ได้เรียนรู้จากการวัดผลครั้งนี้ ที่ไม่มีพูดถึงใน lecture*

> p50 ของ t1-before (229 ms) ดีกว่า t0-large (507 ms) ทั้งที่ระบบแย่ลง เพราะ request ที่ล้มเหลวไม่ถูกนับใน percentile ดังนั้นต้องดู errors และ run time คู่กับ latency เสมอ
