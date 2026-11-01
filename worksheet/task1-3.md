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

> large model ให้ accuracy ที่มากขึ้น แต่ก็แลกมาด้วย latency ที่สูงขึ้นเช่นกัน

## Task 1: Fallback prediction (copy the Task 1 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 1 จาก `results/runs.csv` มาวางที่นี่)

| Run | run time (s) | p50 ms | p95 ms | errors | pending | accuracy |
|---|---|---|---|---|---|---|
| t1-before | 26.7 s | 229 | 411 | 18 | 0 | 93.4% |
| t1-after | 9.6 s | 392 | 1010 | 0 | 28 | 93.0% |

**Q1.1** In `t1-before`, p50 can look *better* than the baseline. Why is p50 misleading here? Which column tells the real story?

*ใน `t1-before` ค่า p50 อาจดู "ดีกว่า" baseline ทำไม p50 จึงทำให้เข้าใจผิดในกรณีนี้ และ column ไหนบอกความจริง*

> p50 คิดจาก request ที่สำเร็จเท่านั้น 18 requests ที่ค้างจน client timeout (10 s) จึงไม่ถูกนับ และ request ที่ค้างยังกิน concurrency slot ไว้ ทำให้ request ที่เหลือแย่ง large model น้อยลงจนดูเร็วขึ้น (229 ms เทียบกับ 507 ms) column ที่บอกความจริงคือ `errors` = 18 ร่วมกับเวลาที่ run ใช้ (26.7 s เทียบกับประมาณ 10 s)

**Q1.2** The hang affects only 10% of large-model calls. Why is it worse for the system than 10% of reviews being slow? Think about what a waiting request holds on to in the review service and the gateway.

*การค้างเกิดกับ calls ไปยัง large model แค่ 10% ทำไมจึงส่งผลต่อระบบแย่กว่าการที่รีวิว 10% ช้าลงเฉยๆ ลองคิดว่า request ที่รออยู่ถือทรัพยากรอะไรไว้ใน review service และ gateway*

> request ที่ค้างถือ connection ไว้ทั้งใน gateway (timeout 60 s) และ review service (ไม่มี timeout) นานถึง 20 s แม้ client จะเลิกรอไปแล้ว ทำให้ request อื่นต้องรอทรัพยากรตามไปด้วย ความล้มเหลวของ inference service จึงลามไปทั้ง chain ไม่ได้จำกัดอยู่แค่ 10% ของรีวิว

**Q1.3** Pending reviews have no label. Suggest what should happen to them, and name the topology from the lecture that does it.

*รีวิวที่ pending ยังไม่มี label ควรจัดการกับรีวิวเหล่านี้อย่างไร และ topology ใดจาก lecture ที่ทำแบบนั้น*

> ควรเก็บรีวิว pending ไว้หรือส่งเข้าคิว แล้วให้ batch job มาติด label ด้วย large model ทีหลัง topology นี้คือ **Asynchronous / batch (T4)** ซึ่งตรงกับ batch processing ใน lecture (inference in bulk, work queue)

**Q1.4** We chose a 1-second timeout. What goes wrong if it is 0.1 s? And if it is 10 s?

*เราเลือก timeout 1 วินาที จะเกิดปัญหาอะไรถ้าตั้งเป็น 0.1 วินาที และถ้าตั้งเป็น 10 วินาที*

> **0.1 s:** สั้นกว่าเวลาปกติของ large model (`t0-large` p50 = 507 ms) รีวิวเกือบทั้งหมดจึงจะกลายเป็น pending แม้โมเดลไม่ได้ค้าง **10 s:** เท่ากับ client timeout ผู้ใช้จึงยังเจอ error และ request ที่ค้างยังถือทรัพยากรไว้นานเหมือนใน Q1.2 ซึ่งผิด QR2 (รอได้ไม่เกิน 1 s)

## Task 2: Cascade (copy the Task 2 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 2 จาก `results/runs.csv` มาวางที่นี่)

| Run | threshold | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|---|
| t2-0.6 | 0.6 | | | | |
| t2-0.7 | 0.7 | | | | |
| t2-0.8 | 0.8 | | | | |
| t2-0.9 | 0.9 | | | | |

**Q2.1** Which threshold would you deploy for CampusEats, and why? Name the quality you give up.

*จะเลือก threshold ใดสำหรับ deploy ใน CampusEats เพราะอะไร และต้องยอมเสีย quality ใดไป*

>

**Q2.2** Look at p50 and p95 for your threshold. Why does the cascade help p50 much more than p95?

*ดูค่า p50 และ p95 ของ threshold ที่เลือก ทำไม cascade จึงช่วย p50 ได้มากกว่า p95 มาก*

>

**Q2.3** The cascade trusts the small model's confidence. Give one situation where that trust is misplaced, and its effect on users.

*cascade เชื่อ confidence ของ small model ยกตัวอย่างหนึ่งสถานการณ์ที่ไม่ควรเชื่อ และผลที่เกิดกับผู้ใช้*

>

## Task 3: Cached predictions (copy the Task 3 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 3 จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls | cache hits |
|---|---|---|---|---|---|
| t3-cache | | | | | |

**Q3.1** Which reviews benefit from the cache, and which never will?

*รีวิวแบบไหนได้ประโยชน์จาก cache และแบบไหนไม่มีวันได้ประโยชน์*

>

**Q3.2** Next week you deploy a better large model. What must happen to the cache, and how would you design for it?

*สัปดาห์หน้าจะ deploy large model ตัวใหม่ที่ดีกว่าเดิม ต้องทำอะไรกับ cache และจะออกแบบรองรับเรื่องนี้อย่างไร*

>

## Debrief

**Q4.** Name one thing you learned from these measurements that the lecture did not cover.

*ยกตัวอย่างหนึ่งสิ่งที่ได้เรียนรู้จากการวัดผลครั้งนี้ ที่ไม่มีพูดถึงใน lecture*

>
