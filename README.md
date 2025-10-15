
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>ระบบสต็อกอุปกรณ์</title>
  <script src="https://unpkg.com/html5-qrcode"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js"></script>

  <style>
  @import url('https://fonts.googleapis.com/css2?family=Prompt&display=swap');

  body {
    font-family: "Prompt", sans-serif;
    margin: 10px;
    background: #f4f6f9;
  }

  .container {
    max-width: 1200px;
    margin: auto;
    background: #fff;
    padding: 20px;
    border-radius: 15px;
    box-shadow: 0 2px 10px rgba(0,0,0,0.1);
  }

  h1 {
    text-align: center;
    color: #333;
    font-size: 28px;
  }

  /* ปุ่มหลัก */
  .btn {
    padding: 12px 18px;
    font-size: 16px;
    border: none;
    background: #1976d2;
    color: white;
    border-radius: 8px;
    cursor: pointer;
    margin: 5px;
    transition: background 0.3s;
    display: inline-block;
  }
  .btn:hover { background: #125aa0; }

  /* ตาราง */
  .table-container {
    overflow-x: auto;
    margin-top: 15px;
  }

  table {
    width: 100%;
    border-collapse: collapse;
    font-size: 16px;
    min-width: 800px;
  }

  th, td {
    border: 1px solid #ccc;
    padding: 10px 6px;
    text-align: center;
  }

  th {
    background: #1976d2;
    color: white;
    font-size: 18px;
  }

  input[type="text"], input[type="number"], select {
    width: 100%;
    max-width: 130px;
    text-align: center;
    font-size: 15px;
    padding: 5px;
    border-radius: 5px;
    border: 1px solid #ccc;
  }

  .low-stock { background: #ffcccc; }  /* สีแดง */
  .high-stock { background: #ccffcc; } /* สีเขียว */
  #reader { width: 100%; max-width: 400px; margin: 20px auto; display: none; }

  /* Responsive ปุ่ม */
  @media (max-width: 768px) {
    h1 { font-size: 24px; }
    .btn {
      font-size: 15px;
      width: 100%;
      display: block;
    }
    .container { padding: 15px; }
    th, td { font-size: 15px; padding: 8px 5px; }
    input[type="text"], input[type="number"], select {
      font-size: 14px;
      width: 100%;
    }
  }

  /* สำหรับปริ้น */
  @media print {
    body * { visibility: hidden; }
    #printArea, #printArea * { visibility: visible; }
    #printArea { position: absolute; left: 0; top: 0; width: 100%; }
    #printArea table { font-size: 16px; }
    #printArea th, #printArea td { padding: 10px 6px; }
  }
  </style>
</head>

<body>
  <div class="container">
    <h1>📦 ระบบสต็อกอุปกรณ์</h1>

    <div style="text-align:center;margin-bottom:15px;">
      <button class="btn" id="start-scan">📸 สแกน QR</button>
      <button class="btn" id="export">📁 บันทึก Excel</button>
      <button class="btn" id="printBtn">🖨️ พิมพ์รายงาน</button>
      <button class="btn" id="addItemBtn">➕ เพิ่มอุปกรณ์</button>
    </div>

    <div id="reader"></div>

    <div class="table-container">
      <table id="stockTable">
        <thead>
          <tr>
            <th>ลำดับ</th>
            <th>ชื่ออุปกรณ์</th>
            <th>คงเหลือ</th>
            <th>หน่วย</th>
            <th>ตำแหน่ง/ห้องสโตร์</th>
            <th>หมายเหตุ</th>
            <th>ขั้นต่ำ</th>
            <th>ต้องสั่งเพิ่ม</th>
            <th>การจัดการ</th>
          </tr>
        </thead>
        <tbody></tbody>
      </table>
    </div>
  </div>

  <!-- สำหรับปริ้น -->
  <div id="printArea" style="display:none;">
    <h2 style="text-align:center;">📦 รายงานสต็อกอุปกรณ์</h2>
    <table id="printTable" style="width:100%; border-collapse:collapse;">
      <thead>
        <tr>
          <th>ลำดับ</th>
          <th>ชื่ออุปกรณ์</th>
          <th>คงเหลือ</th>
          <th>หน่วย</th>
          <th>ตำแหน่ง/ห้องสโตร์</th>
          <th>หมายเหตุ</th>
          <th>ขั้นต่ำ</th>
          <th>ต้องสั่งเพิ่ม</th>
        </tr>
      </thead>
      <tbody></tbody>
    </table>
  </div>

  <script>
  let items = JSON.parse(localStorage.getItem("stockItems")) || [];

  function saveItems() {
    localStorage.setItem("stockItems", JSON.stringify(items));
  }

  function renderTable() {
    const tbody = document.querySelector("#stockTable tbody");
    tbody.innerHTML = "";
    items.forEach((item, index) => {
      const remain = parseInt(item.remain) || 0;
      const min = parseInt(item.min) || 0;
      let rowClass = "";

      if (remain <= min) rowClass = "low-stock";
      else if (remain >= min * 2) rowClass = "high-stock";

      const orderQty = remain <= min ? Math.max(min * 2 - remain, 0) : 0;

      const row = document.createElement("tr");
      row.className = rowClass;
      row.innerHTML = `
        <td>${index + 1}</td>
        <td><input type="text" value="${item.name||''}" onchange="updateItem(${index}, 'name', this.value)"></td>
        <td><input type="text" value="${item.remain||''}" onchange="updateItem(${index}, 'remain', this.value)"></td>
        <td>
          <select onchange="updateItem(${index}, 'unit', this.value)">
            <option value="ชิ้น" ${item.unit==='ชิ้น'?'selected':''}>ชิ้น</option>
            <option value="กล่อง" ${item.unit==='กล่อง'?'selected':''}>กล่อง</option>
            <option value="ลัง" ${item.unit==='ลัง'?'selected':''}>ลัง</option>
            <option value="ม้วน" ${item.unit==='ม้วน'?'selected':''}>ม้วน</option>
            <option value="ใบ" ${item.unit==='ใบ'?'selected':''}>ใบ</option>
            <option value="ตัว" ${item.unit==='ตัว'?'selected':''}>ตัว</option>
            <option value="ดอก" ${item.unit==='ดอก'?'selected':''}>ดอก</option>
            <option value="อื่นๆ" ${item.unit==='อื่นๆ'?'selected':''}>อื่นๆ</option>
          </select>
        </td>
        <td><input type="text" value="${item.location||''}" onchange="updateItem(${index}, 'location', this.value)"></td>
        <td><input type="text" value="${item.note||''}" onchange="updateItem(${index}, 'note', this.value)"></td>
        <td><input type="number" value="${item.min||''}" onchange="updateItem(${index}, 'min', this.value)"></td>
        <td>${orderQty}</td>
        <td><button class="btn" style="background:#e53935" onclick="deleteItem(${index})">ลบ</button></td>
      `;
      tbody.appendChild(row);
    });
  }

  function updateItem(index, key, value) {
    items[index][key] = value;
    saveItems();
    renderTable();
  }

  function addItem(name, remain, unit, location, note, min) {
    const itemName = name || prompt("ชื่ออุปกรณ์:", "");
    if (!itemName) return;
    const itemRemain = remain || prompt("คงเหลือ:", "");
    const itemUnit = unit || prompt("หน่วย:", "ชิ้น");
    const itemLocation = location || prompt("ตำแหน่ง / ห้องสโตร์:", "สโตร์กลาง");
    const itemNote = note || prompt("หมายเหตุ:", "");
    const itemMin = min || prompt("ขั้นต่ำ:", "1");

    items.push({name:itemName, remain:itemRemain, unit:itemUnit, location:itemLocation, note:itemNote, min:itemMin});
    saveItems();
    renderTable();
  }

  function deleteItem(index){
    if(confirm("ลบรายการนี้หรือไม่?")){
      items.splice(index,1);
      saveItems();
      renderTable();
    }
  }

  document.getElementById("addItemBtn").addEventListener("click",()=>addItem());

  document.getElementById("export").addEventListener("click", () => {
    const ws = XLSX.utils.table_to_sheet(document.getElementById("stockTable"));
    const wb = XLSX.utils.book_new();
    XLSX.utils.book_append_sheet(wb, ws, "Stock");
    XLSX.writeFile(wb, "stock_items.xlsx");
  });

  // ปริ้น
  document.getElementById("printBtn").addEventListener("click",()=>{
    const printTbody = document.querySelector("#printTable tbody");
    printTbody.innerHTML = "";

    items.forEach((item, index)=>{
      const remain = parseInt(item.remain)||0;
      const min = parseInt(item.min)||0;
      const orderQty = remain <= min ? Math.max(min*2 - remain, 0) : 0;

      const row = document.createElement("tr");
      row.innerHTML = `
        <td>${index+1}</td>
        <td>${item.name||''}</td>
        <td>${remain}</td>
        <td>${item.unit||''}</td>
        <td>${item.location||''}</td>
        <td>${item.note||''}</td>
        <td>${min}</td>
        <td>${orderQty}</td>
      `;
      printTbody.appendChild(row);
    });

    document.getElementById("printArea").style.display = "block";
    window.print();
    document.getElementById("printArea").style.display = "none";
  });

  // QR Scan
  const html5QrCode = new Html5Qrcode("reader");
  document.getElementById("start-scan").addEventListener("click",()=>{
    document.getElementById("reader").style.display="block";
    Html5Qrcode.getCameras().then(cameras=>{
      if(cameras && cameras.length){
        html5QrCode.start(
          cameras[0].id,{fps:10, qrbox:250},
          decodedText=>{
            const itemName = decodedText;
            const itemRemain = prompt("คงเหลือ:", "");
            const itemUnit = prompt("หน่วย:", "ชิ้น");
            const itemLocation = prompt("ตำแหน่ง / ห้องสโตร์:", "สโตร์กลาง");
            const itemNote = prompt("หมายเหตุ:", "");
            const itemMin = prompt("ขั้นต่ำ:", "1");
            addItem(itemName, itemRemain, itemUnit, itemLocation, itemNote, itemMin);
            alert("เพิ่มอุปกรณ์จาก QR: "+itemName);
            html5QrCode.stop();
            document.getElementById("reader").style.display="none";
          },
          errorMessage=>{ console.log("Scanning...", errorMessage);}
        );
      }
    }).catch(err=>alert("ไม่สามารถเปิดกล้องได้: "+err));
  });

  renderTable();
  </script>
</body>
</html>
