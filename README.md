# <!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>Quiztopia</title>
  <link href="https://fonts.googleapis.com/css2?family=Poppins:wght@400;600&display=swap" rel="stylesheet"/>
  <style>
    :root {
      --primary: #2775ba;
      --correct: #2ecc71;
      --wrong: #e74c3c;
      --bg: #14141e;
      --text: #ffffff;
      --card: rgba(255,255,255,0.05);
    }

    body {
      font-family: 'Poppins', sans-serif;
      background: linear-gradient(135deg, #1f505a, #33334a);
      background-size: 400% 400%;
      animation: gradientBG 20s ease infinite;
      color: var(--text);
      margin: 0;
      padding: 20px;
      display: flex;
      justify-content: center;
      align-items: center;
      min-height: 100vh;
      transition: background 0.5s ease;
    }

    @keyframes gradientBG {
      0% {background-position: 0% 50%}
      50% {background-position: 100% 50%}
      100% {background-position: 0% 50%}
    }

    .quiz-container {
      background-color: var(--card);
      backdrop-filter: blur(10px);
      border-radius: 20px;
      padding: 30px;
      width: 100%;
      max-width: 700px;
      box-shadow: 0 10px 30px rgba(0,0,0,0.5);
      position: relative;
    }

    h1 {
      text-align: center;
      font-size: 2rem;
      margin-bottom: 20px;
      background: linear-gradient(to right, #4facfe, #00f2fe);
      -webkit-background-clip: text;
      color: transparent;
    }

    #question {
      font-size: 18px;
      margin-bottom: 20px;
      text-align: center;
      min-height: 60px;
    }

    .option-btn {
      display: block;
      width: 100%;
      background-color: rgba(255,255,255,0.1);
      color: var(--text);
      padding: 12px;
      margin: 10px 0;
      border-radius: 10px;
      font-size: 16px;
      cursor: pointer;
      border: none;
      transition: all 0.3s ease;
      text-align: left;
    }

    .option-btn:hover {
      transform: translateY(-2px);
      background-color: rgba(255,255,255,0.2);
    }

    #next-btn {
      background: var(--primary);
      color: white;
      padding: 12px 20px;
      border: none;
      border-radius: 10px;
      cursor: pointer;
      font-weight: bold;
      margin: 20px auto 0;
      display: none;
      transition: all 0.3s ease;
    }

    #next-btn:hover {
      transform: translateY(-2px);
      box-shadow: 0 5px 15px rgba(0,0,0,0.3);
    }

    #controls {
      position: absolute;
      top: 15px;
      right: 20px;
      display: flex;
      gap: 10px;
    }

    .control-btn {
      background: none;
      border: none;
      font-size: 20px;
      cursor: pointer;
      color: white;
      transition: all 0.3s ease;
    }

    .control-btn:hover {
      transform: scale(1.2);
    }

    #progress {
      text-align: center;
      font-weight: 600;
      margin-bottom: 10px;
      color: rgba(255,255,255,0.7);
    }

    .light-theme {
      background: linear-gradient(135deg, #ffffff, #cccccc);
      --text: #333333;
      --card: rgba(0,0,0,0.05);
    }

    .light-theme .option-btn {
      background-color: rgba(0,0,0,0.1);
      color: var(--text);
    }

    .light-theme .option-btn:hover {
      background-color: rgba(0,0,0,0.2);
    }

    .light-theme #progress {
      color: rgba(0,0,0,0.7);
    }

    @media (max-width: 600px) {
      .quiz-container {
        padding: 20px;
      }
      
      h1 {
        font-size: 1.5rem;
      }
      
      #question {
        font-size: 16px;
      }
    }
  </style>
</head>
<body>
  <div class="quiz-container">
    <h1>Quiztopia</h1>
    <div id="controls">
      <button id="theme-toggle" class="control-btn">🌓</button>
      <button id="sound-toggle" class="control-btn">🔊</button>
    </div>
    <div id="progress">Solved 0 of 0</div>
    <div id="question">Loading question...</div>
    <div id="options"></div>
    <button id="next-btn">Next</button>
  </div>

  <!-- Audio Elements with working sources -->
  <audio id="correct-sound" preload="auto">
    <source src="https://assets.mixkit.co/sfx/preview/mixkit-correct-answer-tone-2870.mp3" type="audio/mpeg">
  </audio>
  <audio id="wrong-sound" preload="auto">
    <source src="https://assets.mixkit.co/sfx/preview/mixkit-wrong-answer-fail-notification-946.mp3" type="audio/mpeg">
  </audio>

  <script>
    const questionEl = document.getElementById('question');
    const optionsEl = document.getElementById('options');
    const nextBtn = document.getElementById('next-btn');
    const progressEl = document.getElementById('progress');
    const soundToggle = document.getElementById('sound-toggle');
    const themeToggle = document.getElementById('theme-toggle');

    let questionPool = [];
    let currentQuestionIndex = 0;
    let totalSolved = 0;
    let isSoundOn = true;
    let isDark = true;

    // Audio elements with fallback
    const correctSound = document.getElementById('correct-sound');
    const wrongSound = document.getElementById('wrong-sound');

    // Ensure audio can play
    function playSound(sound) {
      if (!isSoundOn) return;
      
      sound.currentTime = 0; // Reset to start
      sound.play().catch(e => {
        console.log("Audio play failed:", e);
        // Fallback for browsers that block audio
        document.body.addEventListener('click', () => sound.play(), { once: true });
      });
    }

    function shuffle(array) {
      for (let i = array.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [array[i], array[j]] = [array[j], array[i]];
      }
      return array;
    }

    async function fetchQuestions(amount = 50) {
      try {
        const res = await fetch(`https://opentdb.com/api.php?amount=${amount}&type=multiple`);
        if (!res.ok) throw new Error('Network response was not ok');
        const data = await res.json();
        if (data.response_code !== 0) throw new Error('API returned no results');
        
        return data.results.map(q => ({
          question: decode(q.question),
          options: shuffle([...q.incorrect_answers, q.correct_answer].map(decode)),
          answer: decode(q.correct_answer)
        }));
      } catch (error) {
        console.error('Error fetching questions:', error);
        questionEl.textContent = "Failed to load questions. Please try again later.";
        return [];
      }
    }

    function decode(html) {
      const txt = document.createElement('textarea');
      txt.innerHTML = html;
      return txt.value;
    }

    function showQuestion() {
      const currentQuestion = questionPool[currentQuestionIndex];
      if (!currentQuestion) return;

      questionEl.textContent = `Q${currentQuestionIndex + 1}: ${currentQuestion.question}`;
      optionsEl.innerHTML = '';
      
      currentQuestion.options.forEach(option => {
        const btn = document.createElement('button');
        btn.className = 'option-btn';
        btn.textContent = option;
        btn.onclick = () => checkAnswer(btn, currentQuestion.answer);
        optionsEl.appendChild(btn);
      });

      nextBtn.style.display = 'none';
      updateProgress();
    }

    function checkAnswer(selectedButton, correctAnswer) {
      const allButtons = document.querySelectorAll('.option-btn');
      allButtons.forEach(btn => btn.disabled = true);
      
      if (selectedButton.textContent === correctAnswer) {
        selectedButton.style.backgroundColor = 'var(--correct)';
        selectedButton.style.color = 'white';
        playSound(correctSound);
      } else {
        selectedButton.style.backgroundColor = 'var(--wrong)';
        selectedButton.style.color = 'white';
        playSound(wrongSound);
        
        // Highlight correct answer
        allButtons.forEach(btn => {
          if (btn.textContent === correctAnswer) {
            btn.style.backgroundColor = 'var(--correct)';
            btn.style.color = 'white';
          }
        });
      }
      
      totalSolved++;
      nextBtn.style.display = 'block';
    }

    nextBtn.addEventListener('click', () => {
      currentQuestionIndex++;
      if (currentQuestionIndex >= questionPool.length - 5) {
        fetchQuestions().then(newQuestions => {
          questionPool = [...questionPool, ...newQuestions];
        });
      }
      showQuestion();
    });

    soundToggle.addEventListener('click', () => {
      isSoundOn = !isSoundOn;
      soundToggle.textContent = isSoundOn ? '🔊' : '🔇';
    });

    themeToggle.addEventListener('click', () => {
      isDark = !isDark;
      if (isDark) {
        document.body.classList.remove('light-theme');
      } else {
        document.body.classList.add('light-theme');
      }
    });

    function updateProgress() {
      progressEl.textContent = `Solved ${totalSolved} of ${currentQuestionIndex + 1}`;
    }

    // Initialize the quiz
    fetchQuestions().then(questions => {
      questionPool = questions;
      showQuestion();
    });

    // Fix for browsers that block audio autoplay
    document.body.addEventListener('click', () => {
      // Just a single interaction to unlock audio
    }, { once: true });
  </script>
</body>
</html>
