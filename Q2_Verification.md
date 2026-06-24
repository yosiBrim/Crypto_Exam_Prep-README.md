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

---

## 3. שבלונת Testbench - קריאה מורכבת מקבצים
כאשר נדרש לכתוב סביבת בדיקה מלאה (ולא Stub) שקוראת מפתח וסיגנלים נכנסים מתוך קובץ (כמו במבחן 23b), נשתמש במנגנון הבא:

```verilog
`timescale 1ns/1ps
`default_nettype none

// ב-TB אין צורך בפורטים (Ports)
module tb_top();

    integer key_check_file;
    integer statusk;
    reg [63:0] key;
    reg [47:0] rk1, rk2, rk3;

    initial begin
        key_check_file = $fopen("key_check_file.txt", "r"); // פתיחה לקריאה
    end

    always @(posedge clk) begin
        if (load) begin
            // קריאת המשתנים מהקובץ, %h כמספר המשתנים.
            statusk = $fscanf(key_check_file, "%h %h %h %h\n", key, rk1, rk2, rk3);
            
            #1; // השהיית סימולציה קלה כדי שהערכים מהקובץ יתעדכנו

            // בדיקה והדפסה במקרה של שגיאה מול התכנון האמיתי (dut)
            if (rk1 != dut.k1)
                $display("Wrong subkey k1: %h instead %h\n", rk1, dut.k1);
        end
    end

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
