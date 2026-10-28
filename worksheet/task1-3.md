# Tutorial 6 worksheet: serving patterns

**Name / ชื่อ: Boriphat Aranyakanon**

**Student ID / รหัสนิสิต: 6631332221**

This is **individual work**: each student completes and submits their own copy, in their own fork.

_งานนี้เป็น**งานเดี่ยว** นิสิตแต่ละคนทำและส่งงานของตัวเอง โดยใช้ forked repository ของตัวเอง_

Copy result rows from `results/runs.csv` (or the load test's output) into the tables. Keep answers short: one to three sentences each. Answer in Thai or English.

ตอนที่ตอบ ให้นิสิตคัดลอกผลจาก `results/runs.csv` (หรือจาก output ของ load test) ลงในตาราง ตอบสั้นๆ ข้อละ 1–3 ประโยค จะตอบเป็นภาษาไทยหรือภาษาอังกฤษก็ได้

## Task 0: Baseline (copy the baseline results from `results/runs.csv` here / copy ผลลัพธ์ของ baseline จาก `results/runs.csv` มาวางที่นี่)

| Run      | p50 ms | p95 ms | accuracy | large calls |
| -------- | ------ | ------ | -------- | ----------- |
| t0-large | 511    | 595    | 93.5%    | 100%        |
| t0-small | 9      | 20     | 72.5%    | 0%          |

**Q0.** What does the large model buy, and what does it cost?

_large model ให้อะไรเพิ่ม และต้องแลกด้วยอะไร_

> เห็นได้ชัดว่า large model ทำให้ accuracy สูงขึ้น แต่แลกมาด้วยเวลาที่ใช้แต่ละ request นานขึ้น

## Task 1: Fallback prediction (copy the Task 1 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 1 จาก `results/runs.csv` มาวางที่นี่)

| Run       | run time (s) | p50 ms | p95 ms | errors | pending | accuracy |
| --------- | ------------ | ------ | ------ | ------ | ------- | -------- |
| t1-before | 28.4         | 227    | 339    | 20     | 0       | 92.8%    |
| t1-after  | 9.2          | 391    | 1009   | 0      | 27      | 93.1%    |

**Q1.1** In `t1-before`, p50 can look _better_ than the baseline. Why is p50 misleading here? Which column tells the real story?

_ใน `t1-before` ค่า p50 อาจดู "ดีกว่า" baseline ทำไม p50 จึงทำให้เข้าใจผิดในกรณีนี้ และ column ไหนบอกความจริง_

> p50 บอกแค่ time per request ที่สำเร็จเท่านั้น request ที่ timeout ซึ่งใช้เวลา 10 วิและ error ไม่ได้ถูกนับ columns ที่บอกความจริงได้ดีกว่าคือ errors และ run time

**Q1.2** The hang affects only 10% of large-model calls. Why is it worse for the system than 10% of reviews being slow? Think about what a waiting request holds on to in the review service and the gateway.

_การค้างเกิดกับ calls ไปยัง large model แค่ 10% ทำไมจึงส่งผลต่อระบบแย่กว่าการที่รีวิว 10% ช้าลงเฉยๆ ลองคิดว่า request ที่รออยู่ถือทรัพยากรอะไรไว้ใน review service และ gateway_

> ด้วยความที่ max connection และ concurrency ถูกกำหนดไว้ ถ้ามี request ที่ค้างมันจะสะสมอยู่ใน slot เรื่อยๆ จนในที่สุดทุก slot ก็เต็ม และ request ที่ค้างไม่ออกจาก slot เพราะว่าเรากำหนด timeout=None ไว้ เมื่อ slot เต็มจะทำให้ request ที่เหลืออีก 90% เข้าไม่ได้ไปด้วย ซึ่งถ้า request แค่ช้าเฉยๆสุดท้ายก็ทำงานเสร็จก็จะคืน slot เอง

**Q1.3** Pending reviews have no label. Suggest what should happen to them, and name the topology from the lecture that does it.

_รีวิวที่ pending ยังไม่มี label ควรจัดการกับรีวิวเหล่านี้อย่างไร และ topology ใดจาก lecture ที่ทำแบบนั้น_

> เก็บไว้ใน queue/broker และจะติด label ทีหลังเป็นชุด เมื่อติด label แล้วอัปเดต status จาก pending เป็น scored ซึ่งคือ topology T4

**Q1.4** We chose a 1-second timeout. What goes wrong if it is 0.1 s? And if it is 10 s?

_เราเลือก timeout 1 วินาที จะเกิดปัญหาอะไรถ้าตั้งเป็น 0.1 วินาที และถ้าตั้งเป็น 10 วินาที_

> timeout เป็น 0.1 จะทำให้ request ส่วนมากรันไม่เสร็จ เพราะว่าค่า p50 อยู่ที่ 0.493 s และถ้า request ทำไม่สำเร็จจะถูกส่งไปเป็น dict pending ส่วน timeout ที่ 10 จะทำให้มีโอกาสเกิด congestion เนื่องจากถ้ามี request ที่รันไม่สำเร็จจำนวนมากแต่ละตัวจะจอง slot ซึ่งถ้า slot เต็ม request ที่เข้ามาใหม่จะต้องต่อ queue ทำให้เกิดปัญหาระบายไม่ทัน

## Task 2: Cascade (copy the Task 2 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 2 จาก `results/runs.csv` มาวางที่นี่)

| Run    | threshold | p50 ms | p95 ms | accuracy | large calls |
| ------ | --------- | ------ | ------ | -------- | ----------- |
| t2-0.6 | 0.6       | 6      | 542    | 91.5%    | 38%         |
| t2-0.7 | 0.7       | 362    | 574    | 93.0%    | 53%         |
| t2-0.8 | 0.8       | 451    | 561    | 93.5%    | 64%         |
| t2-0.9 | 0.9       | 484    | 581    | 93.5%    | 77%         |

**Q2.1** Which threshold would you deploy for CampusEats, and why? Name the quality you give up.

_จะเลือก threshold ใดสำหรับ deploy ใน CampusEats เพราะอะไร และต้องยอมเสีย quality ใดไป_

> เลือก threshold 0.8 เพราะได้ accuracy 93.5 โดยที่ large call น้อยกว่า threshold 0.9 ที่ 64% ต่อ 77% ซึ่งต้องแลกกับ latency/performance(เช่นที่ p50) เพราะว่าที่ threshold = 0.6 ใช้เวลาต่อ request ต่างกันอย่างมาก

**Q2.2** Look at p50 and p95 for your threshold. Why does the cascade help p50 much more than p95?

_ดูค่า p50 และ p95 ของ threshold ที่เลือก ทำไม cascade จึงช่วย p50 ได้มากกว่า p95 มาก_

> p50 ดีขึ้นเพราะมี request ที่ต้องต่อคิวรอ large model น้อยลง ตัวอย่างเช่น มี 200 request จะมี 128 request(64%) วิ่งไปที่ large model และ 72 request วิ่งไปที่ small model นอกจากนี้ request ที่ช้าที่สุด 5% เป็นของ large model เสมอ (large calls 64% > 5%) p95 จึงแทบไม่เปลี่ยน (561 ms ใกล้กับ 595 ms ของ t0-large)

**Q2.3** The cascade trusts the small model's confidence. Give one situation where that trust is misplaced, and its effect on users.

_cascade เชื่อ confidence ของ small model ยกตัวอย่างหนึ่งสถานการณ์ที่ไม่ควรเชื่อ และผลที่เกิดกับผู้ใช้_

> small model ตอบผิดแต่ไม่ส่งต่อให้กับ large model ทำให้ผู้ใช้ได้รับ response ที่ผิด สถานการณ์ตัวอย่างที่ confidence อาจจะส่งผลเสียได้เช่น รีวิวที่บอกว่า "Something was not bad at all" อาจจะเข้าใจผิดว่า "bad" เป็น negative ทั้งๆที่ถ้าดูความหมายโดยรวมเป็น positive

## Task 3: Cached predictions (copy the Task 3 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 3 จาก `results/runs.csv` มาวางที่นี่)

| Run      | p50 ms | p95 ms | accuracy | large calls | cache hits |
| -------- | ------ | ------ | -------- | ----------- | ---------- |
| t3-cache | 6      | 544    | 93.0%    | 38%         | 35%        |

**Q3.1** Which reviews benefit from the cache, and which never will?

_รีวิวแบบไหนได้ประโยชน์จาก cache และแบบไหนไม่มีวันได้ประโยชน์_

> รีวิวที่สั้นได้ประโยชน์มากกว่า เพราะมีโอกาสที่ผู้ใช้จะพิมพ์ประโยคเดียวกันได้มากกว่า รีวิวที่ไม่ได้ประโยชน์เลยคือรีวิวที่เขียนยาว ใส่รายละเอียดเยอะ พูดถึงชื่อเฉพาะหรือความรู้สึกของตนเอง

**Q3.2** Next week you deploy a better large model. What must happen to the cache, and how would you design for it?

_สัปดาห์หน้าจะ deploy large model ตัวใหม่ที่ดีกว่าเดิม ต้องทำอะไรกับ cache และจะออกแบบรองรับเรื่องนี้อย่างไร_

> เนื่องจาก cache ปัจจุบันที่ได้มาจาก model ตัวเก่า จึงไม่สามารถย้ายไปใช้กับ model ตัวใหม่ได้ เพราะฉะนั้นควรใส่ version model ลงใน key ของ cache เพื่อให้สามารถแยก model ที่ใช้ได้

## Debrief

**Q4.** Name one thing you learned from these measurements that the lecture did not cover.

_ยกตัวอย่างหนึ่งสิ่งที่ได้เรียนรู้จากการวัดผลครั้งนี้ ที่ไม่มีพูดถึงใน lecture_

> p50 ไม่สามารถเชื่อถือได้ 100% ใน t1-before ระบบที่มี request ค้างดันมี p50 ดีกว่า
