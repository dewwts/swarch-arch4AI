# Tutorial 6 worksheet: serving patterns

**Name / ชื่อ:** Tinnapat Sittisuwan

**Student ID / รหัสนิสิต:** 6632088321

This is **individual work**: each student completes and submits their own copy, in their own fork.

*งานนี้เป็น**งานเดี่ยว** นิสิตแต่ละคนทำและส่งงานของตัวเอง โดยใช้ forked repository ของตัวเอง*

Copy result rows from `results/runs.csv` (or the load test's output) into the tables. Keep answers short: one to three sentences each. Answer in Thai or English.

ตอนที่ตอบ ให้นิสิตคัดลอกผลจาก `results/runs.csv` (หรือจาก output ของ load test) ลงในตาราง ตอบสั้นๆ ข้อละ 1–3 ประโยค จะตอบเป็นภาษาไทยหรือภาษาอังกฤษก็ได้

## Task 0: Baseline (copy the baseline results from `results/runs.csv` here / copy ผลลัพธ์ของ baseline จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|
| t0-large | 504 | 590 | 93.5% | 100% |
| t0-small | 11 | 25 | 72.5% | 0% |

**Q0.** What does the large model buy, and what does it cost?

*large model ให้อะไรเพิ่ม และต้องแลกด้วยอะไร*

> large ให้ accuracy สูงกว่า small 21 จุดเปอร์เซ็นต์ (93.5% เทียบกับ 72.5%) โดย large ผ่าน QR3 แต่ small ไม่ผ่าน แต่ต้องแลกด้วย p50 ที่สูงกว่า small ประมาณ 46 เท่า (504 เทียบกับ 11 ms) และ p95 สูงกว่าประมาณ 24 เท่า (590 เทียบกับ 25 ms) รวมถึง cost เพราะทุกรีวิว (large calls 100%) ต้องเรียก large model

## Task 1: Fallback prediction (copy the Task 1 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 1 จาก `results/runs.csv` มาวางที่นี่)

| Run | run time (s) | p50 ms | p95 ms | errors | pending | accuracy |
|---|---|---|---|---|---|---|
| t1-before | 25.1 | 260 | 413 | 14 | 0 | 94.1% |
| t1-after | 9.6 | 394 | 1010 | 0 | 28 | 93.0% |

**Q1.1** In `t1-before`, p50 can look *better* than the baseline. Why is p50 misleading here? Which column tells the real story?

*ใน `t1-before` ค่า p50 อาจดู "ดีกว่า" baseline ทำไม p50 จึงทำให้เข้าใจผิดในกรณีนี้ และ column ไหนบอกความจริง*

> loadtest.py คำนวณ p50/p95 จาก request ที่สำเร็จเท่านั้น (`lat = [r["ms"] for r in ok]`) จึงไม่นับ 14 requests ที่ค้างจน timeout นอกจากนี้ p50 ลดจาก 504 เหลือ 260 ms ไม่ใช่เพียงเพราะตัดตัวช้าออก แต่เพราะ requests ที่ hang รอ (sleep) อยู่ก่อนถึง `async with self.capacity` จึงไม่แย่ง Semaphore (limit = 4) คิวภายใน large model สั้นลง requests ที่เหลือแทบไม่ต้องรอคิว latency จึงเข้าใกล้เวลาประมวลผลจริง ตัวเลข p50 จึงดูดีหลอกตา ส่วน column `errors` (14) บอกความจริงว่ามี request ที่ล้มเหลวจากการค้างจน timeout

**Q1.2** The hang affects only 10% of large-model calls. Why is it worse for the system than 10% of reviews being slow? Think about what a waiting request holds on to in the review service and the gateway.

*การค้างเกิดกับ calls ไปยัง large model แค่ 10% ทำไมจึงส่งผลต่อระบบแย่กว่าการที่รีวิว 10% ช้าลงเฉยๆ ลองคิดว่า request ที่รออยู่ถือทรัพยากรอะไรไว้ใน review service และ gateway*

> ระบบเป็น async (FastAPI + httpx.AsyncClient) โดย gateway มี connection pool สูงสุด 200 connections (timeout 60 s) และ review service มี pool 200 เช่นกันแต่ตั้ง timeout=None ทำให้ coroutine และ connection ถูกถือค้างรอต่อไปเรื่อยๆ โดยไม่ถูก reject ทันที ความเสียหายหลักใน lab นี้จึงไม่ได้เกิดจาก pool 200 เต็ม แต่เกิดที่ concurrency slot ของ load test ซึ่งมีเพียง 10 slots แต่ละ request ที่ hang ยึด slot ไว้นาน 10 วินาทีจนกว่า client timeout จะตัดเป็น error เมื่อมี errors 14 ครั้งจึงเสียเวลารวม 14 x 10 = 140 slot-seconds เมื่อหารด้วย 10 slots จะเพิ่ม run time ราว 14 วินาที ตรงกับ run time ที่เพิ่มจากราว 10 s เป็น 25.1 s (t1-before) ต่างจากรีวิวที่ช้าธรรมดาซึ่งคืน slot ให้ request อื่นได้เร็วกว่ามาก

**Q1.3** Pending reviews have no label. Suggest what should happen to them, and name the topology from the lecture that does it.

*รีวิวที่ pending ยังไม่มี label ควรจัดการกับรีวิวเหล่านี้อย่างไร และ topology ใดจาก lecture ที่ทำแบบนั้น*

> ควรมี background worker หรือ batch job มาติด label ให้รีวิว pending ทีหลัง โดยไม่ retry ทันที เพราะถ้า large model กำลังค้างอยู่ การ retry ทันทีจะซ้ำเติมให้ระบบแย่ลง ควรรอให้ upstream ฟื้นตัวก่อนแล้วค่อยลองซ้ำแบบมี backoff หรือรันตามรอบเวลานอกช่วงพีค topology ที่ทำแบบนี้คือ T4 Asynchronous / batch ซึ่งแยกการรับรีวิวออกจากการติด label ทำให้รีวิวไม่หาย (ตรงกับ FR4: t1-after มี errors = 0 และ pending = 28) และไม่บล็อก request ของผู้ใช้

**Q1.4** We chose a 1-second timeout. What goes wrong if it is 0.1 s? And if it is 10 s?

*เราเลือก timeout 1 วินาที จะเกิดปัญหาอะไรถ้าตั้งเป็น 0.1 วินาที และถ้าตั้งเป็น 10 วินาที*

> ถ้าตั้ง 0.1 s รีวิวปกติที่ไม่ได้ hang จะ timeout เกือบทั้งหมด เพราะ large model ตอบปกติราว 500 ms (t0-large p50 = 504) และแม้ไม่มีคิวก็ใช้อย่างน้อยประมาณ 140 ms (200 ms x 0.7) จึงไม่ทัน 0.1 s ผลคือ pending พุ่งเกือบเต็ม 200 และแทบไม่มีรีวิวได้ label (accuracy จะเป็น n/a เพราะวัดเฉพาะรีวิวที่ได้ label) ทั้งที่ large model ไม่ได้มีปัญหา ถ้าตั้ง 10 s จะผิด QR2 ซึ่งให้รอได้ไม่เกิน 1 วินาที (เกิน 10 เท่า) ผู้ใช้ที่เจอ request ค้างต้องรอนานเหมือนใน t1-before และ request นั้นยึด concurrency slot ไว้นาน ทำให้ run time รวมช้าลง

## Task 2: Cascade (copy the Task 2 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 2 จาก `results/runs.csv` มาวางที่นี่)

| Run | threshold | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|---|
| t2-0.6 | 0.6 | 5 | 545 | 91.5% | 38% |
| t2-0.7 | 0.7 | 378 | 578 | 93.0% | 53% |
| t2-0.8 | 0.8 | 505 | 703 | 93.5% | 64% |
| t2-0.9 | 0.9 | 476 | 650 | 93.5% | 77% |

**Q2.1** Which threshold would you deploy for CampusEats, and why? Name the quality you give up.

*จะเลือก threshold ใดสำหรับ deploy ใน CampusEats เพราะอะไร และต้องยอมเสีย quality ใดไป*

> เลือก threshold = 0.7 เพื่อซื้อ safety margin ให้ QR3 ที่ 200 รีวิว threshold 0.6 ได้ accuracy 91.5% ห่างเกณฑ์ 90% เพียง 3 รีวิว เสี่ยงหลุดเกณฑ์ได้ง่ายถ้าข้อมูลจริงใน production ต่างจากชุดทดสอบ ขณะที่ 0.7 ได้ 93.0% มี buffer ถึง 6 รีวิว (3 จุดเปอร์เซ็นต์) สิ่งที่เสียเมื่อเทียบกับ 0.6 คือ large calls เพิ่มขึ้น 15 จุดเปอร์เซ็นต์ (38% เป็น 53%) ซึ่งหมายถึง cost ที่สูงขึ้น และ p50 ช้าลง (378 ms เทียบกับ 5 ms)

**Q2.2** Look at p50 and p95 for your threshold. Why does the cascade help p50 much more than p95?

*ดูค่า p50 และ p95 ของ threshold ที่เลือก ทำไม cascade จึงช่วย p50 ได้มากกว่า p95 มาก*

> ที่ threshold 0.7 small model ตอบเองได้ 47% (อันดับ 1–94 จาก 200) รีวิวที่ตรง p50 (อันดับที่ 100) จึงเป็นรีวิวที่ถูกส่งต่อไป large model กลุ่มแรกๆ ไม่ใช่ small ต่างจาก 0.6 ที่ small ตอบเอง 62% ทำให้ p50 อยู่ในกลุ่ม small ล้วนจึงเหลือ 5 ms ส่วน t2-0.7 ได้ p50 = 378 ms ซึ่งยังเร็วกว่า baseline (504 ms) เพราะ large model รับงานเหลือ 53% คิวหน้า large สั้นลง ส่วน p95 ลดลงแค่ 12 ms (590 เป็น 578) เพราะรีวิวช้าสุด 5% คือกลุ่มที่ small ไม่มั่นใจ ต้องผ่าน small ก่อนแล้วจึงรอ large ซึ่งไม่มีทางเร็วกว่า large ล้วน ได้ประโยชน์เพียงเรื่องคิวที่โล่งขึ้นเท่านั้น

**Q2.3** The cascade trusts the small model's confidence. Give one situation where that trust is misplaced, and its effect on users.

*cascade เชื่อ confidence ของ small model ยกตัวอย่างหนึ่งสถานการณ์ที่ไม่ควรเชื่อ และผลที่เกิดกับผู้ใช้*

> ตัวอย่างคือ "Honestly the salad is not cold." small model (bag of words) ตอบ negative ด้วย confidence 0.702 ซึ่งเฉียดผ่านเกณฑ์ 0.7 จึงไม่ถูกส่งต่อ ทั้งที่ label จริงเป็น positive และ large model ตอบถูก เจ้าของร้านจึงเห็นคำชมกลายเป็นคำตำหนิ ในชุดทดสอบ 500 รีวิว small ผิดแบบมั่นใจ (>= 0.7) 13 เคส แต่ 12 เคส large ก็ผิดเหมือนกัน (ข้อจำกัดของข้อมูล/label ไม่ใช่ความเสียหายจาก cascade) เหลือเพียงเคสนี้ที่ cascade แพ้ large ซึ่งสอดคล้องกับ accuracy ที่ลดจาก 93.5% เป็น 93.0% (0.5 จุดเปอร์เซ็นต์ หรือราว 1 รีวิวจาก 200) แสดงว่า threshold เพียงอย่างเดียวดักเคสที่ small มั่นใจเกินจริงไม่ได้

## Task 3: Cached predictions (copy the Task 3 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 3 จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls | cache hits |
|---|---|---|---|---|---|
| t3-cache | 5 | 559 | 93.0% | 38% | 35% |

**Q3.1** Which reviews benefit from the cache, and which never will?

*รีวิวแบบไหนได้ประโยชน์จาก cache และแบบไหนไม่มีวันได้ประโยชน์*

> ที่ได้ประโยชน์คือรีวิวที่ข้อความซ้ำกัน: กลุ่ม FREQUENT (ข้อความสั้นๆ เช่น "Delicious!") ซ้ำ 48 จาก 56 ครั้ง และกลุ่มที่เหลือบางส่วนก็ถูกสุ่มซ้ำจาก test.csv (27 จาก 144 ครั้ง) รวมราว 75 hits ในการจำลอง ส่วน t3-cache วัดจริงได้ 35% (ต่างเล็กน้อยเพราะ requests ที่ซ้ำกันและเข้ามาพร้อมกันอาจยังไม่ทันถูกเก็บ) โดย key() ที่แปลงเป็นตัวพิมพ์เล็กและยุบช่องว่างทำให้ "Delicious!" กับ "  delicious! " เป็นรายการเดียวกัน แต่ไม่ตัดเครื่องหมายวรรคตอน ส่วนที่ไม่ได้ประโยชน์คือข้อความที่ไม่เคยถูกส่งซ้ำ หรือต่างกันที่คำหรือเครื่องหมาย (key ต่างกัน) accuracy ที่ยัง 93.0% เท่า t2-0.7 บอกว่า cache คืนคำตอบเดิมของโมเดลทั้งถูกและผิด จึงไม่เปลี่ยนคุณภาพ แต่ช่วยลด large calls จาก 53% เหลือ 38% และ p50 จาก 378 เหลือ 5 ms ขณะที่ p95 ลดแค่ 578 เป็น 559 ms เพราะรีวิวที่ miss ยังต้องไปถึง large model

**Q3.2** Next week you deploy a better large model. What must happen to the cache, and how would you design for it?

*สัปดาห์หน้าจะ deploy large model ตัวใหม่ที่ดีกว่าเดิม ต้องทำอะไรกับ cache และจะออกแบบรองรับเรื่องนี้อย่างไร*

> คำตอบที่อยู่ใน cache มาจากโมเดลเวอร์ชันเก่า (cache เก็บผลลัพธ์สุดท้ายของ strategy ทั้งที่มาจาก small และ large) ถ้า deploy โมเดลใหม่แต่ cache ยังอยู่ รีวิวที่ hit จะยังได้คำตอบเก่า (stale predictions) ทำให้ accuracy ไม่ดีขึ้นตามที่คาดหวังจากโมเดลใหม่ จึงต้องล้าง cache หรือออกแบบให้ของเก่าถูกข้ามอัตโนมัติ เช่น ใส่ model version เข้าไปใน cache key (model_version:normalized_text) หรือกำหนด TTL โดยไม่ต้องล้างด้วยมือ trade-off คือหลัง deploy cache จะเย็น (hit rate เป็น 0% ชั่วคราว) large calls จึงเพิ่มขึ้นและ p50 ช้าลงจนกว่า cache จะสะสมคำตอบใหม่

## Debrief

**Q4.** Name one thing you learned from these measurements that the lecture did not cover.

*ยกตัวอย่างหนึ่งสิ่งที่ได้เรียนรู้จากการวัดผลครั้งนี้ ที่ไม่มีพูดถึงใน lecture*

> สิ่งที่ไม่คาดคิดคือ p50 ของ t1-before ลดฮวบจาก 504 เหลือ 260 ms จนดูดีกว่า baseline ทั้งที่ระบบกำลังมีปัญหา เป็นภาพลวงตาจาก 2 ปัจจัยรวมกัน คือ request ที่ค้างกลายเป็น error จึงไม่ถูกนำมาคิดใน latency percentiles และการที่มันค้างอยู่ก่อนแย่ง semaphore ช่วยลดคิวของ request ปกติที่เหลือ ทำให้เห็นว่าการดู latency โดดๆ โดยไม่ดู error rate ควบคู่กันอาจแปลผลประสิทธิภาพผิดได้อย่างสิ้นเชิง
