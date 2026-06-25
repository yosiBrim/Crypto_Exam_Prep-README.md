# 🛡️ ענף 2: טכניקות אימות ווריפיקציה (שאלה 2 במבחן)

**מהות השאלה:** הוכחת הבנה בסימולציות לחומרה, וכתיבת קוד Verilog בסביבת בדיקה. השאלה לרוב תדרוש להסביר מדוע נדרשת סימולציית Gate Level, ולכתוב סביבת בדיקה (Testbench) או מודול דמה (Stub).

---

## 1. שאלות תיאוריה חרושות (Gate Level & Stubs)
במבחנים חוזרת השאלה מדוע חובה לבצע סימולציית Gate Level ולא מספיק להסתפק בסימולציית RTL רגילה. אלו שתי התשובות שצריך לרשום:
* **שינוי קידוד:** במהלך הסינתזה, כלי הסינתזה (כמו Vivado) עלול לשנות את קידוד מכונת המצבים (FSM) לקידוד שונה ממה שנכתב בקוד המקור (למשל מבינארי ל-One-Hot). רק סימולציית Gate Level תוכל לוודא את הקידוד והתזמונים בפועל.
* **קופסאות שחורות (IP Catalog):** מודולים שנוצרו אוטומטית (כמו זכרונות SBOX או FIFO מבוסס BRAM) אינם מגיעים עם קוד Verilog מקורי שניתן לסמלץ. סימולציית Gate Level היא הדרך היחידה להוכיח שהקוד שנוצר עובד כשורה ומשקף את ההשהיות הפיזיות.

---

## 2. שבלונת Stub למודול זיכרון (SBOX)
כאשר המרצה מבקש לכתוב Stub לרכיב כמו `blk_mem_gen_0`, נשתמש בלוגיקה סינכרונית פשוטה המבוססת על משפט `case` שממפה כתובות לנתונים מתוך הטבלה שבשאלה.

```verilog
`default_nettype none

module blk_mem_gen_0 (
    input  wire       clka,
    input  wire [3:0] addra, // רוחב הכתובת לפי השאלה
    output reg  [3:0] douta  // חובה reg כי זה בתוך always
);

    always @(posedge clka) begin
        case (addra)
            // מיפוי לפי הטבלה במבחן
            0:  douta <= 4'hc;
            1:  douta <= 4'h5;
            2:  douta <= 4'h6;
            // ... יש להמשיך את כל הערכים עד הסוף
            15: douta <= 4'h2;
            default: douta <= 4'h0; // מניעת Latch
        endcase
    end
endmodule
```

## 2. כללי הברזל לכתיבת Testbench
* **כלל ה-reg וה-wire ב-Testbench:**
  * כניסות ל-DUT (הקופסה הנבדקת) מוגדרות כ-`reg` (כי אנחנו מייצרים/דוחפים להן ערכים).
  * יציאות מה-DUT מוגדרות כ-`wire` (כי אנחנו רק קוראים מהן).
* **עץ ההחלטות (איזו שבלונה לבחור במבחן?):**
  * **האם יש ל-DUT סיגנלי `start` ו-`ready`?**
    * **כן:** המערכת סדרתית ולוקח לה זמן לחשב. **-> בחר בשבלונה 3ב (עם FSM).**
    * **לא:** המערכת קומבינטורית (כמו SBOX או מודול Loop Unrolled). **-> בחר בשבלונה 3א (קריאה רציפה).**

---

## 3א. שבלונת Testbench למערכת קומבינטורית (ללא start/ready)
**שימוש:** כאשר ה-DUT מחזיר תשובה באופן מיידי ללא עיכוב של עשרות שעונים וללא סיגנלי בקרה. (דוגמה: SBOX או Key Schedule במוד Loop Unrolled).

```verilog
`timescale 1ns/1ps
`default_nettype none

module tb_combinatorial();

    integer data_file_in;
    integer statusD;

    // 1. הגדרת סיגנלים (יש להתאים רוחב לפי השרטוט בשאלה!)
    reg  [55:0]  key_in;
    reg  [767:0] ref_out;     // התוצאה הצפויה מהקובץ
    wire [767:0] actual_out;  // התוצאה בפועל מהמודול

    // 2. מחולל שעון
    reg clk = 0;
    always #10 clk = ~clk;

    // 3. תהליך הבדיקה המרכזי
    initial begin
        data_file_in = $fopen("test_vectors.txt", "r"); // התאם שם קובץ
    end

    // קריאה מהקובץ בעליית שעון
    always @(posedge clk) begin
        if (!$feof(data_file_in)) begin
            // התאם את ה-%h לפי כמות המשתנים בשורה
            statusD = $fscanf(data_file_in, "%h %h\n", key_in, ref_out);
        end else begin
            $fclose(data_file_in);
            #100 $finish;
        end
    end

    // מוניטור ההשוואה (רץ מיד עם קריאת הנתון)
    always @(posedge clk) begin
        if (actual_out == ref_out)
            $display("Time=%8t | Out=%h | PASS\n", $time, actual_out);
        else
            $display("Time=%8t | Out=%h | Expected=%h | FAIL\n", $time, actual_out, ref_out);
    end

    // 4. מופע הרכיב (DUT)
    target_module_name dut_inst (
        .key(key_in),
        .k_out(actual_out)
    );

endmodule
```

---

## 3ב. שבלונת Testbench למערכת סדרתית (מודל FSM של שטרו)
**שימוש:** כאשר המערכת דורשת פרוטוקול "לחיצת יד" (Handshake) - מתן פולס `start` והמתנה לעליית סיגנל ה-`ready`.

```verilog
`timescale 1ns/1ps
`default_nettype none

module tb_sequential_fsm();

    // 1. מצבי מכונת הבדיקה
    localparam IDLE     = 2'b00;
    localparam START    = 2'b01;
    localparam WAIT4RDY = 2'b10;

    reg [1:0] state;
    integer data_file_in, statusD;

    // 2. סיגנלים (להתאים לפי השאלה!)
    reg  clk = 0, reset = 0, start = 0;
    reg  [127:0] plaintext, key, ref_out;
    wire [127:0] ciphertext;
    wire ready;

    always #10 clk = ~clk;

    initial begin
        reset = 1; #50; reset = 0;
        data_file_in = $fopen("test_vectors.txt", "r");
    end

    // 3. הזרקת נתונים וניהול start/ready
    always @(posedge clk) begin
        if (reset) begin
            state <= IDLE;
            start <= 1'b0;
        end else begin
            case (state)
                IDLE: begin
                    state <= START;
                    start <= 1'b1; // פולס התחלה
                end
                START: begin
                    state <= WAIT4RDY;
                    start <= 1'b0; // הורדת הפולס

                    if (!$feof(data_file_in)) begin
                        statusD = $fscanf(data_file_in, "%h %h %h\n", key, plaintext, ref_out);
                    end else begin
                        $fclose(data_file_in);
                        $finish;
                    end
                end
                WAIT4RDY: begin
                    if (ready == 1'b1) begin
                        state <= START; // מתחילים סבב חדש כשהרכיב מוכן
                        start <= 1'b1;
                    end else begin
                        state <= WAIT4RDY;
                        start <= 1'b0;
                    end
                end
            endcase
        end
    end

    // 4. מוניטור ההשוואה (הטריק: מתבצע רק כש-ready עולה!)
    always @(posedge ready) begin
        if (ciphertext == ref_out)
            $display("Time=%0t | Out=%032h | PASS", $time, ciphertext);
        else
            $display("Time=%0t | Out=%032h | FAIL", $time, ciphertext);
    end

    // 5. מופע ה-DUT
    crypto_module_name dut_inst (
        .clk(clk), .reset(reset), .start(start),
        .key(key), .data_in(plaintext),
        .data_out(ciphertext), .ready(ready)
    );

endmodule
```
---

## 4. התבנית האולטימטיבית ל-Stub עם השהיות אלגוריתמיות (File I/O)
כאשר נדרש לייצר Stub שמתעלם מהמידע הנכנס, אך **מציג כלפי חוץ השהיות זהות לתכנון האמיתי וקורא תשובות מקובץ** (כמו במבחנים 23a, 19a), משתמשים בתבנית הבאה. 

✅ **צ'קליסט התאמות למבחן:** 1. שנה את ההדקים (שמות ורוחב ביטים) לפי הגדרות המבחן.
2. עדכן את ה-`counter` לגודל שיכול להכיל את ההשהיה (לפחות 6 ביט לספירה מעל 31).
3. שנה את תנאי העצירה ל-`זמן ההשהיה הנדרש פחות 2`.
4. עדכן את שם הקובץ ב-`$fopen`.

```verilog
`timescale 1ns/1ps
`default_nettype none

// ==========================================
// בלוק 1: עטיפת המודל (להעתיק במדויק מהמבחן)
// ==========================================
module generic_algo_stub (
    input  wire          clk,       
    input  wire          reset,     
    input  wire          start,     
    input  wire [511:0]  data_in,   // רוחב קלט משתנה
    output reg  [127:0]  data_out,  // רוחב פלט משתנה
    output reg           ready      
);

    // ==========================================
    // בלוק 2: הגדרת FSM ומונה השהיות
    // ==========================================
    localparam IDLE = 2'd0;
    localparam WAIT = 2'd1;
    localparam DONE = 2'd2;

    reg [1:0] state;
    reg [5:0] counter; // חובה לוודא שרחב מספיק לספירה (6 ביט = עד 63)

    // ==========================================
    // בלוק 3: קריאה מקבצים
    // ==========================================
    integer file_id;
    integer scan_status;

    initial begin
        file_id = $fopen("expected_answers.txt", "r"); // שם הקובץ מהשאלה
    end

    always @(posedge clk or posedge reset) begin
        if (reset) begin
            state    <= IDLE;
            counter  <= 0;
            ready    <= 1'b0;
            data_out <= 0;
        end else begin
            case (state)
                IDLE: begin
                    ready <= 1'b0; 
                    if (start) begin
                        state   <= WAIT;
                        counter <= 0;
                    end
                end

                WAIT: begin
                    // בלוק 4: לוגיקת דמה (השהיה וקריאה במעבר מצב)
                    // החלף את '62' להשהיה הרצויה פחות 2
                    if (counter == 6'd62) begin 
                        state <= DONE;
                        
                        // שליפת הנתון פעם אחת בדיוק
                        if (!$feof(file_id)) begin
                            scan_status = $fscanf(file_id, "%h\n", data_out); 
                        end else begin
                            $finish; // עצירת סימולציה בסיום הקובץ (אם נדרש)
                        end
                    end else begin
                        counter <= counter + 1;
                    end
                end

                DONE: begin
                    ready <= 1'b1; // הנתון מוכן

                    // הישארות במצב או התחלה מחדש
                    if (start) begin
                        state   <= WAIT;
                        counter <= 0;
                        ready   <= 1'b0;
                    end
                end
            endcase
        end
    end

endmodule
```
