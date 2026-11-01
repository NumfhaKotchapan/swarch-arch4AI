# Tutorial 6 worksheet: serving patterns

**Name / ชื่อ:** กชพรรณ เลิศปัญญาโรจน์

**Student ID / รหัสนิสิต:** 6630001321

This is **individual work**: each student completes and submits their own copy, in their own fork.

*งานนี้เป็น**งานเดี่ยว** นิสิตแต่ละคนทำและส่งงานของตัวเอง โดยใช้ forked repository ของตัวเอง*

Copy result rows from `results/runs.csv` (or the load test's output) into the tables. Keep answers short: one to three sentences each. Answer in Thai or English.

ตอนที่ตอบ ให้นิสิตคัดลอกผลจาก `results/runs.csv` (หรือจาก output ของ load test) ลงในตาราง ตอบสั้นๆ ข้อละ 1–3 ประโยค จะตอบเป็นภาษาไทยหรือภาษาอังกฤษก็ได้

## Task 0: Baseline (copy the baseline results from `results/runs.csv` here / copy ผลลัพธ์ของ baseline จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|
| t0-large | 507 | 615 | 93.5% | 100% |
| t0-small | 54 | 121 | 72.5% | 0% |

**Q0.** What does the large model buy, and what does it cost?

*large model ให้อะไรเพิ่ม และต้องแลกด้วยอะไร*

> Large model ให้ accuracy สูงขึ้นจาก 72.5% เป็น 93.5% หรือเพิ่มขึ้น 21 จุดเปอร์เซ็นต์ ทำให้ผ่านข้อกำหนด accuracy อย่างน้อย 90% ของ CampusEats แต่ต้องแลกกับ latency ที่สูงขึ้น โดย p50 เพิ่มจาก 54 เป็น 507 ms และ p95 เพิ่มจาก 121 เป็น 615 ms รวมถึงต้นทุนการเรียก large model ทุกรีวิว ขณะที่ small model ไม่เรียก large model เลย อย่างไรก็ตาม ในการทดสอบนี้ large model ยังผ่านข้อกำหนดด้านความเร็ว เพราะ p95 ต่ำกว่า 1,000 ms

## Task 1: Fallback prediction (copy the Task 1 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 1 จาก `results/runs.csv` มาวางที่นี่)

| Run | run time (s) | p50 ms | p95 ms | errors | pending | accuracy |
|---|---|---|---|---|---|---|
| t1-before | 34.5 | 258 | 430 | 27 | 0 | 92.5% |
| t1-after | 9.9 | 399 | 1048 | 0 | 24 | 93.2% |

**Q1.1** In `t1-before`, p50 can look *better* than the baseline. Why is p50 misleading here? Which column tells the real story?

*ใน `t1-before` ค่า p50 อาจดู "ดีกว่า" baseline ทำไม p50 จึงทำให้เข้าใจผิดในกรณีนี้ และ column ไหนบอกความจริง*

>ค่า p50 ของ t1-before ลดจาก baseline 507 เหลือ 258 ms แต่ไม่ได้หมายความว่าระบบดีขึ้น เพราะเครื่องมือคำนวณ latency เฉพาะ requests ที่สำเร็จ จึงไม่นับ requests ที่ค้างจนล้มเหลว คอลัมน์ errors บอกปัญหาได้ชัดกว่า โดยมีถึง 27 จาก 200 requests ล้มเหลว และเวลารวมเพิ่มจาก 10.2 เป็น 34.5 วินาที

**Q1.2** The hang affects only 10% of large-model calls. Why is it worse for the system than 10% of reviews being slow? Think about what a waiting request holds on to in the review service and the gateway.

*การค้างเกิดกับ calls ไปยัง large model แค่ 10% ทำไมจึงส่งผลต่อระบบแย่กว่าการที่รีวิว 10% ช้าลงเฉยๆ ลองคิดว่า request ที่รออยู่ถือทรัพยากรอะไรไว้ใน review service และ gateway*

>Requests ที่ค้างยังถือ connections และหน่วยความจำสำหรับงานที่รออยู่ ทั้งใน review service และ gateway ทำให้ทรัพยากรสำหรับรับ requests ใหม่ลดลง นอกจากนี้ load test ส่งพร้อมกันได้เพียง 10 requests เมื่อบางรายการค้างจึงเหลือช่องส่งงานน้อยลง ทำให้การประมวลผลทั้งชุดช้าลงด้วย

**Q1.3** Pending reviews have no label. Suggest what should happen to them, and name the topology from the lecture that does it.

*รีวิวที่ pending ยังไม่มี label ควรจัดการกับรีวิวเหล่านี้อย่างไร และ topology ใดจาก lecture ที่ทำแบบนั้น*

>ควรเก็บ pending reviews ไว้ แล้วให้ background worker หรือ batch job นำไปติด label เมื่อ inference service กลับมาพร้อมใช้งาน พร้อมอัปเดตสถานะเป็น scored วิธีนี้ตรงกับ T4: Asynchronous / batch ซึ่งแยกการรับรีวิวออกจากการติด label เพื่อให้ผู้ใช้ส่งรีวิวได้แม้โมเดลมีปัญหา

**Q1.4** We chose a 1-second timeout. What goes wrong if it is 0.1 s? And if it is 10 s?

*เราเลือก timeout 1 วินาที จะเกิดปัญหาอะไรถ้าตั้งเป็น 0.1 วินาที และถ้าตั้งเป็น 10 วินาที*

>ถ้าตั้งเป็น 0.1 วินาที จะสั้นกว่าเวลาประมวลผลปกติของ large model ที่ประมาณ 0.2 วินาที ทำให้ requests ที่โมเดลทำงานปกติถูกเปลี่ยนเป็น pending จำนวนมาก หากตั้งเป็น 10 วินาที requests ที่ค้างจะถือทรัพยากรนาน ผิดข้อกำหนดที่ review service รอได้ไม่เกิน 1 วินาที และเสี่ยงที่ client จะ timeout ก่อนรับผล fallback

## Task 2: Cascade (copy the Task 2 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 2 จาก `results/runs.csv` มาวางที่นี่)

| Run | threshold | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|---|
| t2-0.6 | 0.6 | 57 | 481 | 91.5% | 38% |
| t2-0.7 | 0.7 | 362 | 588 | 93.0% | 53% |
| t2-0.8 | 0.8 | 438 | 590 | 93.5% | 64% |
| t2-0.9 | 0.9 | 477 | 607 | 93.5% | 77% |

**Q2.1** Which threshold would you deploy for CampusEats, and why? Name the quality you give up.

*จะเลือก threshold ใดสำหรับ deploy ใน CampusEats เพราะอะไร และต้องยอมเสีย quality ใดไป*

>เลือก threshold 0.6 เพราะ accuracy เท่ากับ 91.5% ผ่านเกณฑ์อย่างน้อย 90% และมี p95 เท่ากับ 481 ms ซึ่งต่ำกว่า 1 วินาที โดยเรียก large model เพียง 38% ของ requests จึงลดจำนวนการเรียกได้ประมาณ 62% เมื่อเทียบกับ baseline ต้องยอมเสีย accuracy บางส่วน โดยลดจาก 93.5% ของ large model เหลือ 91.5% หรือ 2 จุดเปอร์เซ็นต์ เพื่อแลกกับความเร็วและต้นทุนที่ลดลง

**Q2.2** Look at p50 and p95 for your threshold. Why does the cascade help p50 much more than p95?

*ดูค่า p50 และ p95 ของ threshold ที่เลือก ทำไม cascade จึงช่วย p50 ได้มากกว่า p95 มาก*

>ที่ threshold 0.6 ประมาณ 62% ของ requests ใช้คำตอบจาก small model ได้เลย ทำให้ request ตรงกลางอยู่ในกลุ่มที่ตอบเร็ว และ p50 ลดจาก baseline 507 เหลือ 57 ms ส่วน requests ที่ช้าที่สุดยังต้องผ่าน small model แล้วรอ large model รวมถึงรอคิวใช้งาน จึงทำให้ p95 ลดจาก 615 เหลือ 481 ms ซึ่งลดลงน้อยกว่า p50 มาก

**Q2.3** The cascade trusts the small model's confidence. Give one situation where that trust is misplaced, and its effect on users.

*cascade เชื่อ confidence ของ small model ยกตัวอย่างหนึ่งสถานการณ์ที่ไม่ควรเชื่อ และผลที่เกิดกับผู้ใช้*

>Small model อาจให้ confidence สูงแต่ตอบผิดเมื่อพบประโยคปฏิเสธ เช่น “not bad” เพราะการดูคำเดี่ยวอาจให้น้ำหนักกับคำว่า “bad” จนตีความเป็น negative หาก confidence ผ่าน threshold ระบบจะใช้คำตอบนั้นโดยไม่ส่งต่อให้ large model ทำให้เจ้าของร้านเห็นรีวิวเชิงบวกเป็นเชิงลบ และอาจเข้าใจความพึงพอใจของลูกค้าผิด

## Task 3: Cached predictions (copy the Task 3 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 3 จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls | cache hits |
|---|---|---|---|---|---|
| t3-cache | 14 | 503 | 91.5% | 31% | 36% |

**Q3.1** Which reviews benefit from the cache, and which never will?

*รีวิวแบบไหนได้ประโยชน์จาก cache และแบบไหนไม่มีวันได้ประโยชน์*

>รีวิวที่มีข้อความซ้ำกับรายการที่ยังอยู่ใน cache จะได้ประโยชน์ เพราะนำผลเดิมมาตอบได้ทันทีโดยไม่ต้องเรียกโมเดล รวมถึงข้อความที่ต่างกันเฉพาะตัวพิมพ์เล็ก–ใหญ่หรือช่องว่างส่วนเกิน เพราะระบบปรับข้อความให้เป็นรูปแบบเดียวกันก่อนค้นหา ส่วนรีวิวที่มีข้อความใหม่ไม่ซ้ำกับรายการใดใน cache จะไม่ได้ประโยชน์ในการเรียกครั้งนั้น และหากข้อความไม่เคยถูกส่งซ้ำเลยก็จะไม่มีโอกาสเกิด cache hit

**Q3.2** Next week you deploy a better large model. What must happen to the cache, and how would you design for it?

*สัปดาห์หน้าจะ deploy large model ตัวใหม่ที่ดีกว่าเดิม ต้องทำอะไรกับ cache และจะออกแบบรองรับเรื่องนี้อย่างไร*

>ควรทำให้ผลจากโมเดลรุ่นเก่าใน cache ใช้งานไม่ได้ เพื่อให้รีวิวได้รับการทำนายด้วยโมเดลที่ปรับปรุงแล้ว โดยอาจล้าง cache เมื่อ deploy หรือใส่ model version เป็นส่วนหนึ่งของ cache key สำหรับระบบ cascade ควรให้ key ครอบคลุมเวอร์ชันของทั้งสองโมเดล รวมถึง strategy และ threshold เพื่อป้องกันการใช้ผลเก่าหลังเปลี่ยนการตั้งค่าการทำนาย

## Debrief

**Q4.** Name one thing you learned from these measurements that the lecture did not cover.

*ยกตัวอย่างหนึ่งสิ่งที่ได้เรียนรู้จากการวัดผลครั้งนี้ ที่ไม่มีพูดถึงใน lecture*

>จากการทดลองได้เรียนรู้ว่า cache อาจช่วยลด p50 ได้มาก โดยไม่ได้ทำให้ p95 ลดลงตามไปด้วยเสมอ ในผล เมื่อเปิด cache ที่ threshold 0.6 ค่า p50 ลดจาก 57 เป็น 14 ms แต่ p95 เพิ่มจาก 481 เป็น 503 ms แม้มี cache hits 36% จึงเห็นว่าต้องประเมินทั้ง latency ของผู้ใช้ทั่วไปและกลุ่มที่รอนาน รวมถึงทดลองซ้ำก่อนสรุปผลต่อ p95 เพราะ requests ที่ไม่พบใน cache ยังต้องเรียกโมเดลและอาจรอคิว
