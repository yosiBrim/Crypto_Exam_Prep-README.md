# 🌿 ענף 1: ממשקים, דפי נתונים (Datasheets) וניתוח ביצועים (שאלה 1 במבחן)

**מהות השאלה:** בשאלה זו יוצג דף נתונים (Datasheet) באנגלית של רכיב קריפטוגרפי. המטרה היא:
1. לכתוב "קליפת" מודול (Interface) ב-Verilog.
2. לחשב מדדים הנדסיים כמו קצב העברת נתונים (Throughput) או זמן ריצה.

---

## 1. שיטת ה"רדאר": 4 הנתונים לחילוץ (סדר פעולות למבחן)
אל תקרא את כל הטקסט! במבחן, רפרף על הדף וחפש אך ורק את 4 הנתונים הבאים לפי הסדר:

* **שלב 1 (למודול): שם הרכיב ורשימת ה-I/O.** חפש שרטוט קופסה עם חצים (Block Diagram) או טבלה בשם `Pin Description`. העתק את שמות הסיגנלים והרוחב שלהם (למשל `[31:0]`) אחד לאחד.
* **שלב 2 (לחישובים): תדר מקסימלי ($f_{max}$).** חפש טבלה בשם `Performance` או `Synthesis Results`. חפש את העמודה של `MHz` והוצא משם את התדר עבור הכרטיס המבוקש.
* **שלב 3 (לחישובים): גודל בלוק המידע (Block Size).** חפש בטקסט את המילים `block size` או `message`. (למשל: "512-bit message block"). זה יהיה המונה בנוסחה שלך.
* **שלב 4 (לחישובים): מחזורי שעון לעיבוד (Clock Cycles).** חפש בטקסט את המילה `cycles`. (למשל: "performed in 66 clock cycles"). זה המחנה בנוסחה.

---

## 2. כתיבת קליפת המודול (Verilog Interface)
**חוק ברזל:** חובה תמיד להתחיל עם `default_nettype none`. אין צורך בלוגיקה פנימית, רק כניסות ויציאות לפי מה שחילצת בשלב 1.

**תבנית קוד אופיינית:**
```verilog
`default_nettype none // חובה!

module crypto_core_name ( // השם משלב 1
    input  wire         clk, //
    input  wire         rst_n, //
    input  wire         load, //
    input  wire [31:0]  din, // לפי הרוחב בטבלה
    
    output wire         ready, //
    output wire [127:0] dout //
);
    // This answer only requires I/Os specifications
endmodule //
```

---

## 3. נוסחאות חומרה וביצועים (Performance Calculations)
השתמש בנתונים שחילצת בשלבים 2, 3 ו-4.

### א. חישוב זמן ביצוע (Execution Time)
* **הנוסחה:** $Execution\_Time = Cycles \cdot T = \frac{Cycles}{Frequency}$
* **דוגמה:** רכיב דורש 180,000 מחזורי שעון בתדר 212.27MHz. 
  $Time = 180,000 \cdot \frac{1}{212.27 \cdot 10^6} = 84.797 [ms]$.

### ב. חישוב קצב העברת נתונים (Throughput)
* **הנוסחה:** $Throughput = \frac{Block\_Size\_in\_Bits}{Execution\_Time\_per\_Block}$.

### 💡 קיצור דרך חישובי (The "Bits per Cycle" Ratio)
כדי לחסוך המרות כשיש טבלת תדרים:
1. חשב יחס קבוע: `Ratio = Block_Size / Clock_Cycles`.
2. הכפל את היחס בתדר של הכרטיס המבוקש: `Throughput = Ratio * Frequency`.
*(זהירות: באלגוריתם כמו RSA, יש לחשב את ה-Clock Cycles ידנית בעזרת Square and Multiply לפני שמבצעים את היחס).*
