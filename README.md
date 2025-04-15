<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>Lucien's Dashboard</title>
  <script src="https://cdn.tailwindcss.com"></script>
  <script src="https://cdn.jsdelivr.net/npm/canvas-confetti@1.6.0/dist/confetti.browser.min.js"></script>
  <style>
    .casino-price {
      background: linear-gradient(45deg, #FFD700, #FFA500);
      color: #fff;
      padding: 0.25rem 0.75rem;
      border-radius: 9999px;
      font-weight: bold;
      font-size: 1.125rem;
      box-shadow: 0 0 10px #FFA500, 0 0 20px #FF4500 inset;
      text-shadow: 0 0 5px #FF6347;
    }
    .task-card {
      transition: transform 0.3s ease, box-shadow 0.3s ease, border-color 0.5s ease;
      touch-action: pan-y;
      animation: bounceIn 0.6s ease;
      border: 2px solid;
    }
    .task-card:hover {
      transform: scale(1.02);
      box-shadow: 0 0 20px rgba(255, 165, 0, 0.7);
    }
    @keyframes bounceIn {
      0% { transform: scale(0.9); opacity: 0; }
      60% { transform: scale(1.1); opacity: 1; }
      100% { transform: scale(1); }
    }
    .swipe-complete {
      animation: bounceOut 0.6s forwards ease-in-out;
    }
    @keyframes bounceOut {
      0% { transform: translateX(0) scale(1); opacity: 1; }
      50% { transform: translateX(40%) scale(1.15); }
      100% { transform: translateX(120%) scale(0.9); opacity: 0; }
    }
  </style>
</head>
<body class="bg-gradient-to-br from-gray-900 via-black to-gray-800 text-white min-h-screen p-6">
  <div class="max-w-lg mx-auto">
    <div class="mb-4 text-center">
      <select id="viewSelector" class="bg-gray-800 text-white border border-yellow-500 p-2 rounded">
        <option value="tasks">🎯 Todays Tasks</option>
        <option value="setter">🛠️ Task Setter</option>
      </select>
    </div>

    <div id="taskView">
      <div class="mb-6">
        <div class="flex justify-between items-center mb-1">
          <h2 class="text-sm font-bold text-yellow-400">XP Progress</h2>
          <div class="text-sm text-gray-400">
            <span id="earned">$0</span> / <span id="total">$0</span>
          </div>
        </div>
        <div class="w-full h-4 bg-gray-700 rounded-full overflow-hidden">
          <div id="xpBar" class="h-full bg-yellow-500 transition-all duration-300" style="width: 0%"></div>
        </div>
      </div>

      <div class="mb-4 text-center">
        <h1 class="text-4xl font-bold text-yellow-400 animate-pulse">🎯 Todays thumbnails 🎯</h1>
        <p class="text-sm text-gray-400 mt-2">Time Worked: <span id="timeWorked">0</span>/<span id="totalTime">0</span>h</p>
      </div>

      <div class="mb-4 text-center">
        <button id="undoButton" class="bg-red-600 hover:bg-red-700 transition text-white py-2 px-4 rounded-lg shadow-md" disabled>Undo Last Swipe</button>
      </div>

      <div id="taskList"></div>
    </div>

    <div id="setterView" class="hidden">
      <div class="text-center">
        <h2 class="text-2xl font-bold text-yellow-400 mb-4">🛠️ Task Setter</h2>
        <p class="text-sm text-gray-400">Lucien can customize his tasks here (interface under construction).</p>
      </div>
    </div>

    <audio id="notificationSound" src="https://voca.ro/19758gWJ2KyR.mp3" preload="auto"></audio>

    <script>
      let originalIndex = {};
      const tasks = [
        { name: "QuickCut", time: 2, amount: 80, deadline: "2025-04-15T19:00:00Z" },
        { name: "DailyHacks", time: 1, amount: 45, deadline: "2025-04-15T17:00:00Z" },
        { name: "ViralVision", time: 4, amount: 120, deadline: "2025-04-15T21:00:00Z" },
        { name: "Footcrunch", time: 5, amount: 150, deadline: "2025-04-15T22:00:00Z" },
        { name: "Stokes Twins", time: 3, amount: 60, deadline: "2025-04-16T12:00:00Z" },
        { name: "BuzzWorld", time: 2, amount: 90, deadline: "2025-04-15T20:00:00Z" }
      ];

      const taskList = document.getElementById('taskList');
      const undoButton = document.getElementById('undoButton');
      const totalTaskTime = tasks.reduce((sum, task) => sum + task.time, 0);
      let totalMoney = tasks.reduce((sum, task) => sum + task.amount, 0);
      let earnedMoney = 0;
      let workedTime = 0;
      let lastCompletedTask = null;

      document.getElementById('total').textContent = `$${totalMoney}`;
      document.getElementById('totalTime').textContent = totalTaskTime;

      tasks.sort((a, b) => new Date(a.deadline) - new Date(b.deadline));
      tasks.forEach((task, index) => {
        originalIndex[task.name] = index;
        createTaskCard(task, index);
      });

      function createTaskCard(task, index) {
        const percent = (task.time / totalTaskTime) * 100;
        const now = new Date();
        const deadline = new Date(task.deadline);
        const hoursUntil = (deadline - now) / (1000 * 60 * 60);
        let red = Math.min(255, Math.max(0, 255 - hoursUntil * 25));
        let green = Math.min(255, Math.max(0, hoursUntil * 25));
        const borderColor = `rgb(${Math.round(red)}, ${Math.round(green)}, 0)`;

        const card = document.createElement('div');
        card.className = "task-card bg-gray-800 p-5 rounded-xl mb-6 flex justify-between items-center";
        card.style.borderColor = borderColor;
        card.setAttribute('data-time', task.time);
        card.setAttribute('data-amount', task.amount);
        card.setAttribute('data-deadline', task.deadline);
        card.setAttribute('data-name', task.name);
        card.setAttribute('draggable', 'true');

        const predictedXP = Math.floor((task.amount / totalMoney) * 100);

        card.innerHTML = `
          <div class="flex-1">
            <h2 class="text-xl font-semibold">${task.name}</h2>
            <div class="flex items-center gap-3 mt-2">
              <span class="casino-price" title="Adds ~${predictedXP}% XP">$${task.amount}</span>
              <span class="text-sm text-gray-300">Due in: <span class="text-orange-400 font-bold countdown"></span></span>
            </div>
          </div>
          <div class="relative w-14 h-14 ml-4">
            <svg class="absolute top-0 left-0 w-full h-full" viewBox="0 0 36 36">
              <path fill="none" stroke="#4B5563" stroke-width="4" d="M18 2.0845 a 15.9155 15.9155 0 0 1 0 31.831 a 15.9155 15.9155 0 0 1 0 -31.831" />
              <path class="progress-ring" fill="none" stroke="#10B981" stroke-width="4" stroke-dasharray="${percent}, 100" d="M18 2.0845 a 15.9155 15.9155 0 0 1 0 31.831 a 15.9155 15.9155 0 0 1 0 -31.831" />
            </svg>
            <div class="absolute inset-0 flex items-center justify-center text-xs text-white">${task.time}h</div>
          </div>`;

        let startX = 0;
        card.addEventListener('dragstart', e => startX = e.clientX);
        card.addEventListener('dragend', e => {
          const offset = e.clientX - startX;
          if (offset > 100) {
            earnedMoney += task.amount;
            workedTime += task.time;
            document.getElementById('earned').textContent = `$${earnedMoney}`;
            document.getElementById('timeWorked').textContent = workedTime;
            const xp = (earnedMoney / totalMoney) * 100;
            document.getElementById('xpBar').style.width = `${xp}%`;

            card.classList.add('swipe-complete');
            document.getElementById('notificationSound').play();
            confetti({ particleCount: 150, spread: 70, origin: { y: 0.6 } });
            setTimeout(() => card.remove(), 600);

            lastCompletedTask = task;
            undoButton.disabled = false;
          }
        });

        const countdownEl = card.querySelector('.countdown');
        function updateCountdown() {
          const now = new Date();
          const diff = deadline - now;
          const hoursLeft = Math.max(0, Math.floor(diff / (1000 * 60 * 60)));
          countdownEl.textContent = `${hoursLeft}H`;
        }
        updateCountdown();
        setInterval(updateCountdown, 60000);

        const allCards = document.querySelectorAll('.task-card');
        if (index >= allCards.length) taskList.appendChild(card);
        else taskList.insertBefore(card, allCards[index]);
      }

      undoButton.addEventListener('click', () => {
        if (lastCompletedTask) {
          earnedMoney -= lastCompletedTask.amount;
          workedTime -= lastCompletedTask.time;
          document.getElementById('earned').textContent = `$${earnedMoney}`;
          document.getElementById('timeWorked').textContent = workedTime;
          const xp = (earnedMoney / totalMoney) * 100;
          document.getElementById('xpBar').style.width = `${xp}%`;

          tasks.push(lastCompletedTask);
          tasks.sort((a, b) => new Date(a.deadline) - new Date(b.deadline));
          taskList.innerHTML = '';
          tasks.forEach((task, index) => {
            originalIndex[task.name] = index;
            createTaskCard(task, index);
          });

          lastCompletedTask = null;
          undoButton.disabled = true;
        }
      });

      document.getElementById('viewSelector').addEventListener('change', function () {
        const taskView = document.getElementById('taskView');
        const setterView = document.getElementById('setterView');
        if (this.value === 'tasks') {
          taskView.classList.remove('hidden');
          setterView.classList.add('hidden');
        } else {
          taskView.classList.add('hidden');
          setterView.classList.remove('hidden');
        }
      });
    </script>
  </div>
</body>
</html>
