<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8" />
  <title>カレンダー家計簿</title>
  <style>
    body {
      font-family: system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
      margin: 0;
      padding: 16px;
      background: #f5f5f7;
    }
    h1 {
      margin-top: 0;
      font-size: 20px;
    }
    .header {
      display: flex;
      align-items: center;
      gap: 8px;
      margin-bottom: 8px;
    }
    button {
      cursor: pointer;
    }
    .month-label {
      font-weight: bold;
      font-size: 16px;
      margin: 0 8px;
    }
    .tabs {
      display: flex;
      gap: 8px;
      margin: 12px 0;
    }
    .tab {
      padding: 6px 12px;
      border-radius: 999px;
      border: 1px solid #ccc;
      background: #fff;
      font-size: 13px;
    }
    .tab.active {
      background: #007aff;
      color: #fff;
      border-color: #007aff;
    }
    .calendar {
      display: grid;
      grid-template-columns: repeat(7, 1fr);
      gap: 4px;
      margin-bottom: 12px;
    }
    .weekday {
      text-align: center;
      font-size: 11px;
      color: #666;
    }
    .day-cell {
      background: #fff;
      border-radius: 6px;
      min-height: 60px;
      padding: 4px;
      font-size: 11px;
      border: 1px solid #e0e0e0;
      position: relative;
      cursor: pointer;
    }
    .day-cell.today {
      border-color: #007aff;
    }
    .day-number {
      font-weight: bold;
      font-size: 12px;
    }
    .day-total {
      position: absolute;
      bottom: 4px;
      right: 4px;
      font-size: 11px;
      color: #007aff;
    }
    .panel {
      background: #fff;
      border-radius: 8px;
      padding: 10px;
      border: 1px solid #e0e0e0;
      margin-bottom: 12px;
    }
    .panel-title {
      font-size: 13px;
      font-weight: bold;
      margin-bottom: 6px;
    }
    .form-row {
      display: flex;
      gap: 8px;
      margin-bottom: 6px;
      flex-wrap: wrap;
    }
    .form-row label {
      font-size: 12px;
    }
    input[type="number"],
    input[type="text"],
    select {
      font-size: 12px;
      padding: 3px 4px;
    }
    .list {
      font-size: 12px;
      max-height: 160px;
      overflow-y: auto;
    }
    .list-item {
      display: flex;
      justify-content: space-between;
      border-bottom: 1px solid #eee;
      padding: 2px 0;
    }
    .summary-row {
      display: flex;
      justify-content: space-between;
      font-size: 12px;
      margin-bottom: 2px;
    }
    .small {
      font-size: 11px;
      color: #666;
    }
  </style>
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
</head>
<body>
  <h1>カレンダー家計簿</h1>

  <div class="header">
    <button id="prevMonthBtn">◀</button>
    <div class="month-label" id="monthLabel"></div>
    <button id="todayBtn">今月</button>
    <button id="nextMonthBtn">▶</button>
  </div>

  <div class="tabs">
    <button class="tab active" data-tab="calendar">月間カレンダー</button>
    <button class="tab" data-tab="daily">日別詳細</button>
    <button class="tab" data-tab="weekly">週ごと集計</button>
  </div>

  <!-- 月間カレンダー -->
  <div id="calendarView">
    <div class="calendar" id="calendarHeader">
      <div class="weekday">日</div>
      <div class="weekday">月</div>
      <div class="weekday">火</div>
      <div class="weekday">水</div>
      <div class="weekday">木</div>
      <div class="weekday">金</div>
      <div class="weekday">土</div>
    </div>
    <div class="calendar" id="calendarBody"></div>
  </div>

  <!-- 日別詳細 -->
  <div id="dailyView" style="display:none;">
    <div class="panel">
      <div class="panel-title" id="dailyTitle">日別詳細</div>
      <div class="small" id="dailyDateLabel"></div>
      <div class="list" id="dailyList"></div>
      <div class="summary-row">
        <span>合計支出</span>
        <span id="dailyTotal">0 円</span>
      </div>
    </div>
  </div>

  <!-- 週ごと集計 -->
  <div id="weeklyView" style="display:none;">
    <div class="panel">
      <div class="panel-title">週ごと集計</div>
      <div class="list" id="weeklyList"></div>
    </div>
  </div>

  <!-- 入力フォーム（共通） -->
  <div class="panel">
    <div class="panel-title" id="formTitle">選択した日への入力</div>
    <div class="small" id="formDateLabel"></div>
    <div class="form-row">
      <label>金額：
        <input type="number" id="amountInput" min="0" />
      </label>
      <label>区分：
        <select id="categoryInput">
          <option value="食費">食費</option>
          <option value="日用品">日用品</option>
          <option value="交通">交通</option>
          <option value="娯楽">娯楽</option>
          <option value="固定費">固定費</option>
          <option value="その他">その他</option>
        </select>
      </label>
      <label>収支：
        <select id="typeInput">
          <option value="expense">支出</option>
          <option value="income">収入</option>
        </select>
      </label>
    </div>
    <div class="form-row">
      <label style="flex:1;">メモ：
        <input type="text" id="memoInput" style="width:100%;" />
      </label>
    </div>
    <button id="addRecordBtn">追加</button>
  </div>

  <script>
    // ---- データ構造 ----
    // records[YYYY-MM-DD] = [{amount, category, type, memo}]
    // ---- データ保存 ----
    const records = JSON.parse(localStorage.getItem("kakeiboRecords")) || {};

    function saveRecords() {
      localStorage.setItem(
        "kakeiboRecords",
        JSON.stringify(records)
      );
    }

    // ---- 日付ユーティリティ ----
    function formatDateKey(date) {
      const y = date.getFullYear();
      const m = String(date.getMonth() + 1).padStart(2, "0");
      const d = String(date.getDate()).padStart(2, "0");
      return `${y}-${m}-${d}`;
    }

    function parseDateKey(key) {
      const [y, m, d] = key.split("-").map(Number);
      return new Date(y, m - 1, d);
    }

    // ---- 状態 ----
    let currentDate = new Date();
    let selectedDate = new Date();

    // ---- DOM取得 ----
    const monthLabel = document.getElementById("monthLabel");
    const calendarBody = document.getElementById("calendarBody");
    const dailyView = document.getElementById("dailyView");
    const weeklyView = document.getElementById("weeklyView");
    const calendarView = document.getElementById("calendarView");

    const dailyTitle = document.getElementById("dailyTitle");
    const dailyDateLabel = document.getElementById("dailyDateLabel");
    const dailyList = document.getElementById("dailyList");
    const dailyTotal = document.getElementById("dailyTotal");

    const weeklyList = document.getElementById("weeklyList");

    const formTitle = document.getElementById("formTitle");
    const formDateLabel = document.getElementById("formDateLabel");
    const amountInput = document.getElementById("amountInput");
    const categoryInput = document.getElementById("categoryInput");
    const typeInput = document.getElementById("typeInput");
    const memoInput = document.getElementById("memoInput");
    const addRecordBtn = document.getElementById("addRecordBtn");

    const tabs = document.querySelectorAll(".tab");

    // ---- カレンダー描画 ----
    function renderCalendar() {
      calendarBody.innerHTML = "";
      const year = currentDate.getFullYear();
      const month = currentDate.getMonth();
      monthLabel.textContent = `${year}年 ${month + 1}月`;

      const firstDay = new Date(year, month, 1);
      const lastDay = new Date(year, month + 1, 0);
      const startWeekday = firstDay.getDay();
      const daysInMonth = lastDay.getDate();

      // 空白セル
      for (let i = 0; i < startWeekday; i++) {
        const cell = document.createElement("div");
        calendarBody.appendChild(cell);
      }

      const todayKey = formatDateKey(new Date());

      for (let day = 1; day <= daysInMonth; day++) {
        const cell = document.createElement("div");
        cell.className = "day-cell";

        const date = new Date(year, month, day);
        const key = formatDateKey(date);

        if (key === todayKey) {
          cell.classList.add("today");
        }

        const num = document.createElement("div");
        num.className = "day-number";
        num.textContent = day;
        cell.appendChild(num);

        // その日の合計支出を表示
        const dayRecords = records[key] || [];
        const totalExpense = dayRecords
          .filter(r => r.type === "expense")
          .reduce((sum, r) => sum + r.amount, 0);

        if (totalExpense > 0) {
          const totalEl = document.createElement("div");
          totalEl.className = "day-total";
          totalEl.textContent = `▲${totalExpense.toLocaleString()}円`;
          cell.appendChild(totalEl);
        }

        cell.addEventListener("click", () => {
          selectedDate = date;
          updateFormDateLabel();
          renderDailyView();
          renderWeeklyView();
          // タブを日別に切り替えると分かりやすい
          setActiveTab("daily");
        });

        calendarBody.appendChild(cell);
      }
    }

    // ---- 日別詳細描画 ----
    function renderDailyView() {
      const key = formatDateKey(selectedDate);
      const list = records[key] || [];
      dailyList.innerHTML = "";

      dailyTitle.textContent = "日別詳細";
      dailyDateLabel.textContent = key;

      let totalExpense = 0;

      list.forEach((r, idx) => {
        const item = document.createElement("div");
        item.className = "list-item";

        const left = document.createElement("span");
        left.textContent = `${r.type === "expense" ? "支出" : "収入"} / ${r.category} / ${r.amount.toLocaleString()}円`;

        const right = document.createElement("div");

        const memoSpan = document.createElement("span");
        memoSpan.textContent = r.memo || "";

        const deleteBtn = document.createElement("button");
        deleteBtn.textContent = "削除";
        deleteBtn.style.marginLeft = "8px";
        deleteBtn.style.fontSize = "10px";

        deleteBtn.addEventListener("click", () => {

  if (!confirm("このデータを削除しますか？")) {
    return;
  }

  records[key].splice(idx, 1);

  if (records[key].length === 0) {
    delete records[key];
  }

  saveRecords();

  renderCalendar();
  renderDailyView();
  renderWeeklyView();
});

right.appendChild(memoSpan);
right.appendChild(deleteBtn);

        item.appendChild(left);
        item.appendChild(right);

        dailyList.appendChild(item);

        if (r.type === "expense") {
          totalExpense += r.amount;
        }
      });

      dailyTotal.textContent = `${totalExpense.toLocaleString()} 円`;
    }

    // ---- 週ごと集計描画 ----
    function renderWeeklyView() {
      weeklyList.innerHTML = "";

      const year = currentDate.getFullYear();
      const month = currentDate.getMonth();
      const firstDay = new Date(year, month, 1);
      const lastDay = new Date(year, month + 1, 0);
      const daysInMonth = lastDay.getDate();

      // 1日から順に週ごとにまとめる
      let weekIndex = 1;
      let start = 1;

      while (start <= daysInMonth) {
        const startDate = new Date(year, month, start);
        const startWeekday = startDate.getDay();
        const end = Math.min(start + (6 - startWeekday), daysInMonth);

        let weekExpense = 0;

        for (let d = start; d <= end; d++) {
          const date = new Date(year, month, d);
          const key = formatDateKey(date);
          const list = records[key] || [];
          list.forEach(r => {
            if (r.type === "expense") {
              weekExpense += r.amount;
            }
          });
        }

        const row = document.createElement("div");
        row.className = "summary-row";

        const label = document.createElement("span");
        label.textContent = `第${weekIndex}週（${month + 1}/${start}〜${month + 1}/${end}）`;

        const value = document.createElement("span");
        value.textContent = `支出合計：${weekExpense.toLocaleString()} 円`;

        row.appendChild(label);
        row.appendChild(value);
        weeklyList.appendChild(row);

        weekIndex++;
        start = end + 1;
      }
    }

    // ---- フォーム関連 ----
    function updateFormDateLabel() {
      const key = formatDateKey(selectedDate);
      formTitle.textContent = "選択した日への入力";
      formDateLabel.textContent = key;
    }

    addRecordBtn.addEventListener("click", () => {
      const amount = Number(amountInput.value);
      if (!amount || amount <= 0) {
        alert("金額を入力してください");
        return;
      }
      const category = categoryInput.value;
      const type = typeInput.value;
      const memo = memoInput.value.trim();

      const key = formatDateKey(selectedDate);
      if (!records[key]) {
        records[key] = [];
      }
      records[key].push({
        amount,
        category,
        type,
        memo
      });

saveRecords();

      amountInput.value = "";
      memoInput.value = "";

      renderCalendar();
      renderDailyView();
      renderWeeklyView();
    });

    // ---- タブ切り替え ----
    function setActiveTab(tabName) {
      tabs.forEach(t => {
        t.classList.toggle("active", t.dataset.tab === tabName);
      });

      calendarView.style.display = tabName === "calendar" ? "block" : "none";
      dailyView.style.display = tabName === "daily" ? "block" : "none";
      weeklyView.style.display = tabName === "weekly" ? "block" : "none";
    }

    tabs.forEach(tab => {
      tab.addEventListener("click", () => {
        setActiveTab(tab.dataset.tab);
      });
    });

    // ---- 月移動 ----
    document.getElementById("prevMonthBtn").addEventListener("click", () => {
      currentDate.setMonth(currentDate.getMonth() - 1);
      renderCalendar();
      renderWeeklyView();
    });

    document.getElementById("nextMonthBtn").addEventListener("click", () => {
      currentDate.setMonth(currentDate.getMonth() + 1);
      renderCalendar();
      renderWeeklyView();
    });

    document.getElementById("todayBtn").addEventListener("click", () => {
      currentDate = new Date();
      selectedDate = new Date();
      updateFormDateLabel();
      renderCalendar();
      renderDailyView();
      renderWeeklyView();
    });

    // ---- 初期化 ----
    (function init() {
      selectedDate = new Date();
      updateFormDateLabel();
      renderCalendar();
      renderDailyView();
      renderWeeklyView();
    })();
  </script>
</body>
</html>
