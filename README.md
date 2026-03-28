package com.thobshop.app

import android.content.ContentValues
import android.content.Context
import android.database.sqlite.SQLiteDatabase
import android.database.sqlite.SQLiteOpenHelper
import android.graphics.Bitmap
import android.graphics.Color
import android.net.Uri
import android.os.Bundle
import android.os.Environment
import android.text.Editable
import android.text.TextWatcher
import android.view.*
import android.widget.*
import androidx.appcompat.app.AppCompatActivity
import androidx.appcompat.app.AppCompatDelegate
import androidx.recyclerview.widget.LinearLayoutManager
import androidx.recyclerview.widget.RecyclerView
import java.io.File
import java.io.FileOutputStream
import java.text.SimpleDateFormat
import java.util.*

// ================== قاعدة البيانات ==================
class ThobShopApp(context: Context) :
    SQLiteOpenHelper(context, "thobshop.db", null, 1) {

    override fun onCreate(db: SQLiteDatabase) {
        db.execSQL("""CREATE TABLE users(id INTEGER PRIMARY KEY AUTOINCREMENT,name TEXT,username TEXT,password TEXT,role TEXT)""")
        db.execSQL("""CREATE TABLE inventory(
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT,color TEXT,size INTEGER,length INTEGER,width INTEGER,neck INTEGER,sleeve INTEGER,
            fabric TEXT,hand TEXT,pockets TEXT,buttons TEXT,zipper TEXT,collar TEXT,
            purchasePrice REAL,salePrice REAL,quantity INTEGER,supplierId INTEGER
        )""")
        db.execSQL("""CREATE TABLE sales(id INTEGER PRIMARY KEY AUTOINCREMENT,productId INTEGER,workerId INTEGER,price REAL,discount REAL,paid REAL,remaining REAL,date TEXT)""")
        db.execSQL("""CREATE TABLE reservations(id INTEGER PRIMARY KEY AUTOINCREMENT,customer TEXT,phone TEXT,productId INTEGER,quantity INTEGER,paid REAL,remaining REAL,date TEXT)""")
        db.execSQL("""CREATE TABLE tailoring(id INTEGER PRIMARY KEY AUTOINCREMENT,customer TEXT,phone TEXT,length INTEGER,width INTEGER,neck INTEGER,sleeve INTEGER,fabric TEXT,color TEXT,hand TEXT,pockets TEXT,price REAL,paid REAL,remaining REAL,deliveryDate TEXT)""")
        db.execSQL("""CREATE TABLE alterations(id INTEGER PRIMARY KEY AUTOINCREMENT,customer TEXT,phone TEXT,type TEXT,price REAL,paid REAL,remaining REAL)""")
        db.execSQL("""CREATE TABLE debts(id INTEGER PRIMARY KEY AUTOINCREMENT,name TEXT,phone TEXT,amount REAL,paid REAL,remaining REAL,type TEXT)""")
        db.execSQL("""CREATE TABLE expenses(id INTEGER PRIMARY KEY AUTOINCREMENT,name TEXT,amount REAL,date TEXT)""")
    }

    override fun onUpgrade(db: SQLiteDatabase, oldVersion: Int, newVersion: Int) {}

    // ================== العمليات الأساسية ==================
    fun addUser(name:String, username:String, password:String, role:String){
        val db = writableDatabase
        val values = ContentValues()
        values.put("name",name)
        values.put("username",username)
        values.put("password",password)
        values.put("role",role)
        db.insert("users",null,values)
    }

    fun addThob(
        name:String,color:String,size:Int,length:Int,width:Int,neck:Int,sleeve:Int,
        fabric:String,hand:String,pockets:String,buttons:String,zipper:String,collar:String,
        purchasePrice:Double,salePrice:Double,quantity:Int,supplierId:Int
    ){
        val db = writableDatabase
        val values = ContentValues()
        values.put("name",name)
        values.put("color",color)
        values.put("size",size)
        values.put("length",length)
        values.put("width",width)
        values.put("neck",neck)
        values.put("sleeve",sleeve)
        values.put("fabric",fabric)
        values.put("hand",hand)
        values.put("pockets",pockets)
        values.put("buttons",buttons)
        values.put("zipper",zipper)
        values.put("collar",collar)
        values.put("purchasePrice",purchasePrice)
        values.put("salePrice",salePrice)
        values.put("quantity",quantity)
        values.put("supplierId",supplierId)
        db.insert("inventory",null,values)
    }

    fun getAllThobs(): MutableList<Map<String, Any>> {
        val db = readableDatabase
        val cursor = db.rawQuery("SELECT * FROM inventory", null)
        val list = mutableListOf<Map<String, Any>>()
        if (cursor.moveToFirst()){
            do {
                val map = mutableMapOf<String, Any>()
                map["id"] = cursor.getInt(cursor.getColumnIndexOrThrow("id"))
                map["name"] = cursor.getString(cursor.getColumnIndexOrThrow("name"))
                map["color"] = cursor.getString(cursor.getColumnIndexOrThrow("color"))
                map["size"] = cursor.getInt(cursor.getColumnIndexOrThrow("size"))
                map["quantity"] = cursor.getInt(cursor.getColumnIndexOrThrow("quantity"))
                map["salePrice"] = cursor.getDouble(cursor.getColumnIndexOrThrow("salePrice"))
                list.add(map)
            } while(cursor.moveToNext())
        }
        cursor.close()
        return list
    }

    // ================== المبيعات المتقدمة ==================
    fun addSaleAdvanced(productId: Int, workerId: Int, price: Double, discount: Double, paid: Double) {
        val finalPrice = price - discount
        val remaining = finalPrice - paid
        val db = writableDatabase
        val values = ContentValues()
        values.put("productId", productId)
        values.put("workerId", workerId)
        values.put("price", price)
        values.put("discount", discount)
        values.put("paid", paid)
        values.put("remaining", remaining)
        values.put("date", System.currentTimeMillis())
        db.insert("sales", null, values)

        // تحديث المخزون
        val cursor = readableDatabase.rawQuery("SELECT quantity FROM inventory WHERE id=?", arrayOf(productId.toString()))
        if (cursor.moveToFirst()) {
            val currentQty = cursor.getInt(cursor.getColumnIndexOrThrow("quantity"))
            val newQty = if (currentQty > 0) currentQty - 1 else 0
            val cv = ContentValues()
            cv.put("quantity", newQty)
            writableDatabase.update("inventory", cv, "id=?", arrayOf(productId.toString()))
        }
        cursor.close()
    }

    // ================== الحجز ==================
    fun addReservationAdvanced(customer: String, phone: String, productId: Int, quantity: Int, paid: Double){
        val db = writableDatabase
        val values = ContentValues()
        values.put("customer", customer)
        values.put("phone", phone)
        values.put("productId", productId)
        values.put("quantity", quantity)
        values.put("paid", paid)
        values.put("remaining", quantity*0.0)
        values.put("date", System.currentTimeMillis())
        db.insert("reservations", null, values)
    }

    // ================== التفصيل ==================
    fun addTailoringAdvanced(customer: String, phone: String, length: Int, width: Int, neck: Int, sleeve: Int, fabric: String, color: String, hand: String, pockets: String, price: Double, paid: Double, deliveryDate: String){
        val remaining = price - paid
        val db = writableDatabase
        val values = ContentValues()
        values.put("customer", customer)
        values.put("phone", phone)
        values.put("length", length)
        values.put("width", width)
        values.put("neck", neck)
        values.put("sleeve", sleeve)
        values.put("fabric", fabric)
        values.put("color", color)
        values.put("hand", hand)
        values.put("pockets", pockets)
        values.put("price", price)
        values.put("paid", paid)
        values.put("remaining", remaining)
        values.put("deliveryDate", deliveryDate)
        db.insert("tailoring", null, values)
    }

    // ================== الديون ==================
    fun addDebtAdvanced(name: String, phone: String, amount: Double, type: String){
        val db = writableDatabase
        val values = ContentValues()
        values.put("name", name)
        values.put("phone", phone)
        values.put("amount", amount)
        values.put("paid", 0.0)
        values.put("remaining", amount)
        values.put("type", type)
        db.insert("debts", null, values)
    }

    fun payDebt(id: Int, amount: Double){
        val cursor = readableDatabase.rawQuery("SELECT paid, remaining FROM debts WHERE id=?", arrayOf(id.toString()))
        if(cursor.moveToFirst()){
            val paidPrev = cursor.getDouble(cursor.getColumnIndexOrThrow("paid"))
            val remainingPrev = cursor.getDouble(cursor.getColumnIndexOrThrow("remaining"))
            val newPaid = paidPrev + amount
            val newRemaining = remainingPrev - amount
            val cv = ContentValues()
            cv.put("paid", newPaid)
            cv.put("remaining", newRemaining)
            writableDatabase.update("debts", cv, "id=?", arrayOf(id.toString()))
        }
        cursor.close()
    }

    // ================== التقارير ==================
    fun dailyReport(): String {
        val cursor = readableDatabase.rawQuery("SELECT * FROM sales WHERE date >= ?", arrayOf(System.currentTimeMillis().toString()))
        var totalIncome = 0.0
        var totalDiscount = 0.0
        while(cursor.moveToNext()){
            val price = cursor.getDouble(cursor.getColumnIndexOrThrow("price"))
            val discount = cursor.getDouble(cursor.getColumnIndexOrThrow("discount"))
            totalIncome += price
            totalDiscount += discount
        }
        cursor.close()
        return "تقرير اليوم: الدخل = $totalIncome ريال، الخصومات = $totalDiscount ريال"
    }

    fun monthlyReport(): String {
        val calendar = Calendar.getInstance()
        val month = calendar.get(Calendar.MONTH)
        val year = calendar.get(Calendar.YEAR)
        val cursor = readableDatabase.rawQuery(
            "SELECT * FROM sales WHERE strftime('%m', date/1000, 'unixepoch')=? AND strftime('%Y', date/1000, 'unixepoch')=?",
            arrayOf((month+1).toString(), year.toString())
        )
        var totalIncome = 0.0
        var totalDiscount = 0.0
        while(cursor.moveToNext()){
            val price = cursor.getDouble(cursor.getColumnIndexOrThrow("price"))
            val discount = cursor.getDouble(cursor.getColumnIndexOrThrow("discount"))
            totalIncome += price
            totalDiscount += discount
        }
        cursor.close()
        return "تقرير الشهر: الدخل = $totalIncome ريال، الخصومات = $totalDiscount ريال"
    }

    // ================== العمال ==================
    fun addWorker(name:String, username:String, password:String){ addUser(name, username, password, "worker") }
    fun getWorkers(): MutableList<Map<String, Any>>{
        val db = readableDatabase
        val cursor = db.rawQuery("SELECT * FROM users WHERE role='worker'", null)
        val list = mutableListOf<Map<String, Any>>()
        if(cursor.moveToFirst()){
            do{
                val map = mutableMapOf<String, Any>()
                map["id"] = cursor.getInt(cursor.getColumnIndexOrThrow("id"))
                map["name"] = cursor.getString(cursor.getColumnIndexOrThrow("name"))
                map["username"] = cursor.getString(cursor.getColumnIndexOrThrow("username"))
                list.add(map)
            } while(cursor.moveToNext())
        }
        cursor.close()
        return list
    }

    // ================== البحث ==================
    fun filterThobs(thobs: MutableList<Map<String, Any>>, query: String): MutableList<Map<String, Any>> {
        return thobs.filter {
            (it["name"] as String).contains(query, true) ||
            (it["color"] as String).contains(query, true) ||
            (it["size"].toString()).contains(query)
        }.toMutableList()
    }

    // ================== الجرد اليومي ==================
    fun dailyInventoryReport(): String {
        val cursor = readableDatabase.rawQuery("SELECT SUM(quantity), SUM(salePrice*quantity) FROM inventory", null)
        var totalQty = 0
        var totalValue = 0.0
        if(cursor.moveToFirst()){
            totalQty = cursor.getInt(0)
            totalValue = cursor.getDouble(1)
        }
        cursor.close()
        return "الجرد اليومي: عدد الثياب = $totalQty، القيمة = $totalValue ريال"
    }
}

// ================== MainActivity ==================
class MainActivity : AppCompatActivity() {
    lateinit var app: ThobShopApp
    lateinit var recyclerView: RecyclerView
    lateinit var adapter: ThobAdapter

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val layout = LinearLayout(this)
        layout.id = 100
        layout.orientation = LinearLayout.VERTICAL

        val title = TextView(this)
        title.text = "لمسات العالمي - المخزون"
        title.textSize = 22f
        layout.addView(title)

        val searchEdit = EditText(this)
        searchEdit.hint = "بحث عن الثوب..."
        layout.addView(searchEdit)

        recyclerView = RecyclerView(this)
        recyclerView.layoutManager = LinearLayoutManager(this)
        layout.addView(recyclerView, LinearLayout.LayoutParams(LinearLayout.LayoutParams.MATCH_PARENT, 0, 1f))

        setContentView(layout)

        app = ThobShopApp(this)
        app.addThob("ثوب رجالي","أبيض",52,140,30,42,62,"قطن","كبك","يمين","أزرار","سحاب","قلب عادي",5000.0,7000.0,5,1)
        app.addThob("ثوب شبابي","أسود",48,135,28,40,60,"رهيب","سادة","ليس","أزرار","سحاب","عادي",4500.0,6500.0,3,1)

        adapter = ThobAdapter(app.getAllThobs())
        recyclerView.adapter = adapter

        searchEdit.addTextChangedListener(object: TextWatcher{
            override fun afterTextChanged(s: Editable?) {}
            override fun beforeTextChanged(s: CharSequence?, start: Int, count: Int, after: Int) {}
            override fun onTextChanged(s: CharSequence?, start: Int, before: Int, count: Int) {
                val filtered = app.filterThobs(app.getAllThobs(), s.toString())
                adapter.updateList(filtered)
            }
        })
    }
}

// ================== Adapter ==================
class ThobAdapter(private var thobs: MutableList<Map<String, Any>>) : RecyclerView.Adapter<ThobAdapter.ThobViewHolder>() {
    class ThobViewHolder(val layout: LinearLayout) : RecyclerView.ViewHolder(layout){
        val name: TextView = TextView(layout.context)
        val color: TextView = TextView(layout.context)
        val size: TextView = TextView(layout.context)
        val quantity: TextView = TextView(layout.context)
        val price: TextView = TextView(layout.context)
        init {
            layout.orientation = LinearLayout.VERTICAL
            layout.addView(name)
            layout.addView(color)
            layout.addView(size)
            layout.addView(quantity)
            layout.addView(price)
        }
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ThobViewHolder {
        val layout = LinearLayout(parent.context)
        layout.setPadding(20,20,20,20)
        layout.layoutParams = RecyclerView.LayoutParams(RecyclerView.LayoutParams.MATCH_PARENT, RecyclerView.LayoutParams.WRAP_CONTENT)
        return ThobViewHolder(layout)
    }

    override fun onBindViewHolder(holder: ThobViewHolder, position: Int) {
        val item = thobs[position]
        holder.name.text = "اسم الثوب: ${item["name"]}"
        holder.color.text = "اللون: ${item["color"]}"
        holder.size.text = "المقاس: ${item["size"]}"
        holder.quantity.text = "الكمية: ${item["quantity"]}"
        holder.price.text = "سعر البيع: ${item["salePrice"]} ريال"
    }

    override fun getItemCount(): Int = thobs.size
    fun updateList(newList: MutableList<Map<String, Any>>){
        thobs = newList
        notifyDataSetChanged()
    }
}
