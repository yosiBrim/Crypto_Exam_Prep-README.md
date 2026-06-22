# 🛡️ ענף 2: טכניקות אימות ווריפיקציה (שאלה 2 במבחן)

**מהות השאלה:** הוכחת הבנה בסימולציות לחומרה, וכתיבת קוד Verilog בסביבת בדיקה. השאלה לרוב תדרוש להסביר מדוע נדרשת סימולציית Gate Level, ולכתוב סביבת בדיקה (Testbench) או מודול דמה (Stub).

---

## 1. שאלות תיאוריה חרושות (Gate Level & Stubs)
במבחנים חוזרת השאלה מדוע חובה לבצע סימולציית Gate Level ולא מספיק להסתפק בסימולציית RTL רגילה. אלו שתי התשובות שצריך לרשום:
* **שינוי קידוד:** במהלך הסינתזה, כלי הסינתזה (כמו Vivado) עלול לשנות את קידוד מכונת המצבים (FSM) לקידוד שונה ממה שנכתב בקוד המקור. רק סימולציית Gate Level תוכל לוודא את הקידוד בפועל.
* **קופסאות שחורות (IP Catalog):** מודולים שנוצרו אוטומטית (כמו זכרונות SBOX) אינם מגיעים עם קוד Verilog מקורי שניתן לסמלץ. סימולציית Gate Level היא הדרך היחידה להוכיח שהקוד שנוצר עובד כשורה. מודול "דמה" (Stub) משמש כתחליף זמני לפונקציונליות הזו כדי לאפשר קומפילציה.

---

## 2. שבלונת Stub למודול זיכרון (SBOX)
כאשר המרצה מבקש לכתוב Stub לרכיב כמו `blk_mem_gen_0`, נשתמש בלוגיקה סינכרונית פשוטה המבוססת על משפט `case` שממפה כתובות לנתונים מתוך הטבלה שבשאלה.

```verilog
`default_nettype none //

module blk_mem_gen_0 ( //
    input  wire       clka, //
    input  wire [3:0] addra, // רוחב הכתובת לפי השאלה
    output reg  [3:0] douta  // חובה reg כי זה בתוך always
);

    always @(posedge clka) begin //
        case (addra) //
            // מיפוי לפי הטבלה במבחן
            0:  douta <= 4'hc; //
            1:  douta <= 4'h5; //
            2:  douta <= 4'h6; //
            // ... יש להמשיך את כל הערכים עד הסוף
            15: douta <= 4'h2; //
            default: douta <= 4'h0; // מניעת Latch
        endcase //
    end
endmodule //
```

---

## 3. שבלונת Testbench - קריאה מורכבת מקבצים
קריאת נתונים מרובים משורה אחת בקובץ (למשל מפתח ועוד מספר תת-מפתחות) בעזרת `$fscanf`.

```verilog
integer key_check_file; //
integer statusk; //
reg [63:0] key; //
reg [47:0] rk1, rk2, rk3; //

initial begin
    key_check_file = $fopen("key_check_file.txt", "r"); // פתיחה לקריאה
end

always @(posedge clk) begin
    if (load) begin
        // קריאת המשתנים מהקובץ, %h כמספר המשתנים.
        statusk = $fscanf(key_check_file, "%h %h %h %h\n", key, rk1, rk2, rk3); //
        
        #1; // השהיית סימולציה כדי שהערכים יתעדכנו

        // בדיקה והדפסה במקרה של שגיאה
        if (rk1 != dut.k1) //
            $display("Wrong subkey k1: %h instead %h\n", rk1, dut.k1); //
    end
end
```
