To create a **beautiful, modern, accessible, and minimalistic button** with an **animated border gradient** using **Open Props** and the **Cube CSS** methodology, we’ll combine semantic HTML, accessibility best practices, and utility-first CSS using Open Props for tokens and Cube CSS structure.

---

### ✅ Features:

- **Animated gradient border**
    
- **Focus and hover states**
    
- **Accessible (keyboard + screen reader)**
    
- **Cube CSS structure (Composition, Utilities, Blocks, Elements)**
    
- **Uses Open Props tokens for color, spacing, and motion**
    

---

## 💻 Code

### 🧩 HTML

```html
<button class="btn-border-animated" type="button">
  Click Me
</button>
```

---

### 🎨 CSS (with Open Props & Cube CSS)

Make sure you have Open Props included. You can use a CDN:

```html
<link rel="stylesheet" href="https://unpkg.com/open-props" />
```

Then, here’s the SCSS/CSS structure using Cube CSS principles:

```css
/* 1. Utilities - Using Open Props */
@import "https://unpkg.com/open-props/normalize.min.css";
@import "https://unpkg.com/open-props/open-props.min.css";
@import "https://unpkg.com/open-props/animations.min.css";

/* 2. Composition */
:root {
  --btn-radius: var(--radius-2);
  --btn-padding: var(--size-2);
  --btn-font-size: var(--font-size-2);
  --btn-bg: var(--surface-1);
  --btn-text-color: var(--text-1);
  --btn-border-gradient: linear-gradient(45deg, var(--pink-5), var(--indigo-5), var(--cyan-5));
}

/* 3. Block - Button */
.btn-border-animated {
  position: relative;
  display: inline-block;
  padding: var(--btn-padding) var(--size-4);
  font-size: var(--btn-font-size);
  font-weight: var(--font-weight-6);
  color: var(--btn-text-color);
  background-color: var(--btn-bg);
  border: none;
  border-radius: var(--btn-radius);
  cursor: pointer;
  isolation: isolate;
  transition: color 0.3s ease-in-out, background-color 0.3s ease-in-out;
}

/* 4. Pseudo Element for Border */
.btn-border-animated::before {
  content: "";
  position: absolute;
  inset: -2px;
  z-index: -1;
  border-radius: inherit;
  background: var(--btn-border-gradient);
  background-size: 200% 200%;
  animation: gradient-move 3s ease infinite;
  mask: 
    linear-gradient(#fff 0 0) content-box, 
    linear-gradient(#fff 0 0);
  mask-composite: exclude;
  -webkit-mask-composite: destination-out;
  padding: 2px;
}

/* 5. Interactions */
.btn-border-animated:focus-visible {
  outline: 2px solid var(--cyan-6);
  outline-offset: 4px;
}

.btn-border-animated:hover {
  background-color: var(--surface-2);
  color: var(--pink-5);
}

/* 6. Animation */
@keyframes gradient-move {
  0% {
    background-position: 0% 50%;
  }
  50% {
    background-position: 100% 50%;
  }
  100% {
    background-position: 0% 50%;
  }
}
```

* * *

* * *

* * *

```css
/* 1. Utilities - Using Open Props */
@import "https://unpkg.com/open-props/normalize.min.css";
@import "https://unpkg.com/open-props/open-props.min.css";
@import "https://unpkg.com/open-props/animations.min.css";

/* 2. Composition */
:root {
  --card-radius: var(--radius-3);
  --card-padding: var(--size-4);
  --card-gap: var(--size-3);
  --card-bg: var(--surface-1);
  --card-text-color: var(--text-1);
  --card-border-gradient: linear-gradient(45deg, var(--pink-5), var(--indigo-5), var(--cyan-5));
  --btn-padding: var(--size-2);
  --btn-font-size: var(--font-size-2);
  --btn-bg: var(--surface-1);
  --btn-text-color: var(--text-1);
  --btn-hover-bg: var(--surface-2);
  --btn-hover-text-color: var(--pink-5);
  --btn-focus-outline: var(--cyan-6);
}

/* 3. Block - Button */
.btn-border-animated {
  position: relative;
  display: inline-block;
  padding: var(--btn-padding) var(--size-4);
  font-size: var(--btn-font-size);
  font-weight: var(--font-weight-6);
  color: var(--btn-text-color);
  background-color: var(--btn-bg);
  border: none;
  border-radius: var(--btn-radius);
  cursor: pointer;
  isolation: isolate;
  transition: color 0.3s ease-in-out, background-color 0.3s ease-in-out;
}

/* 4. Pseudo Element for Border */
.btn-border-animated::before {
  content: "";
  position: absolute;
  inset: -2px;
  z-index: -1;
  border-radius: inherit;
  background: var(--btn-border-gradient);
  background-size: 200% 200%;
  animation: gradient-move 3s ease infinite;
  mask: 
    linear-gradient(#fff 0 0) content-box, 
    linear-gradient(#fff 0 0);
  mask-composite: exclude;
  -webkit-mask-composite: destination-out;
  padding: 2px;
}

/* 5. Interactions */
.btn-border-animated:focus-visible {
  outline: 2px solid var(--cyan-6);
  outline-offset: 4px;
}

.btn-border-animated:hover {
  background-color: var(--surface-2);
  color: var(--pink-5);
}

/* 6. Animation */
@keyframes gradient-move {
  0% {
    background-position: 0% 50%;
  }
  50% {
    background-position: 100% 50%;
  }
  100% {
    background-position: 0% 50%;
  }
}

/* 7. Layout - Dashboard */
.dashboard {
  display: grid;
  grid-template-columns: 1fr;
  gap: var(--size-5);
  padding: var(--size-5);
  background-color: var(--surface-3);
}

/* 8. Each Dashboard Section */
.dashboard-section {
  background-color: var(--surface-1);
  border-radius: var(--radius-3);
  padding: var(--size-4);
  box-shadow: var(--shadow-2);
  display: flex;
  flex-direction: column;
  gap: var(--size-4);
}

.section-title {
  font-size: var(--font-size-4);
  font-weight: var(--font-weight-6);
  color: var(--text-1);
}

/* 9. Card Grid inside each section */
.card-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(240px, 1fr));
  gap: var(--size-4);
}

/* 10. Reusing previously created card styles */
.card {
  position: relative;
  display: flex;
  flex-direction: column;
  gap: var(--size-3);
  background-color: var(--surface-1);
  padding: var(--size-4);
  border-radius: var(--radius-3);
  box-shadow: var(--shadow-1);
  isolation: isolate;
  overflow: hidden;
}

.card::before {
  content: "";
  position: absolute;
  inset: -2px;
  z-index: -1;
  border-radius: inherit;
  background: linear-gradient(45deg, var(--pink-5), var(--indigo-5), var(--cyan-5));
  background-size: 200% 200%;
  animation: gradient-move 3s ease infinite;
  mask: linear-gradient(#fff 0 0) content-box, linear-gradient(#fff 0 0);
  mask-composite: exclude;
  -webkit-mask-composite: destination-out;
  padding: 2px;
}

.card-title {
  font-size: var(--font-size-3);
  font-weight: var(--font-weight-6);
  color: var(--text-1);
}

.card-description {
  font-size: var(--font-size-2);
  color: var(--text-2);
}

/* 11. Button with icon */
.btn-with-icon {
  display: inline-flex;
  align-items: center;
  gap: var(--size-1);
  padding: var(--size-2) var(--size-4);
  font-size: var(--font-size-2);
  font-weight: var(--font-weight-6);
  border-radius: var(--radius-2);
  border: none;
  background-color: var(--surface-2);
  color: var(--text-1);
  cursor: pointer;
  transition: background-color 0.3s ease, color 0.3s ease;
}

.btn-with-icon:hover {
  background-color: var(--surface-3);
  color: var(--pink-5);
}

.icon-arrow {
  width: 1rem;
  height: 1rem;
  transition: transform 0.3s ease;
}

.btn-with-icon:hover .icon-arrow {
  transform: translateX(4px);
}

/* 12. Block - Card */
.dashboard-footer {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: var(--size-4);
  padding: var(--size-4);
  background-color: var(--surface-2);
  border-top: 1px solid var(--gray-4);
}

/* 13. Console Log Block */
.console-log {
  background-color: var(--surface-1);
  border-radius: var(--radius-2);
  padding: var(--size-3);
  box-shadow: var(--shadow-2);
}

.console-title {
  margin-bottom: var(--size-2);
  font-size: var(--font-size-3);
  color: var(--text-1);
}

.console-messages {
  list-style: none;
  padding: 0;
  margin: 0;
  font-family: monospace;
  color: var(--text-2);
}

.console-messages li {
  margin-bottom: var(--size-1);
}

/* 14. Progress Bar Block */
.progress-section {
  background-color: var(--surface-1);
  border-radius: var(--radius-2);
  padding: var(--size-3);
  box-shadow: var(--shadow-2);
}

.progress-title {
  margin-bottom: var(--size-2);
  font-size: var(--font-size-3);
  color: var(--text-1);
}

.progress-bar-group {
  display: flex;
  flex-direction: column;
  gap: var(--size-2);
}

.progress-item {
  display: flex;
  flex-direction: column;
  gap: var(--size-1);
  font-size: var(--font-size-1);
  color: var(--text-2);
}

/* 15. Custom progress style using Open Props */
progress {
  width: 100%;
  height: 8px;
  appearance: none;
}

progress::-webkit-progress-bar {
  background-color: var(--gray-3);
  border-radius: 100px;
}

progress::-webkit-progress-value {
  background-image: linear-gradient(90deg, var(--green-5), var(--teal-5));
  border-radius: 100px;
  transition: width 0.5s ease;
}

progress::-moz-progress-bar {
  background-image: linear-gradient(90deg, var(--green-5), var(--teal-5));
  border-radius: 100px;
}
```

* * *

* * *

* * *
```html
<button class="btn-border-animated" type="button">
  Click Me
</button>

<main class="dashboard">
  <!-- Repeat this section 6 times -->
  <section class="dashboard-section">
    <header class="section-header">
      <h2 class="section-title">Section Title</h2>
    </header>
    <div class="card-grid">
      <!-- 4 Cards -->
      <article class="card">
        <header class="card-header">
          <h3 class="card-title">Card Title</h3>
          <p class="card-description">Brief description of the card content.</p>
        </header>
        <footer class="card-footer">
          <button class="btn-with-icon">
            Learn More
            <svg class="icon-arrow" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none">
              <path d="M12 2L12 22M12 22L5 15M12 22L19 15" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
            </svg>
          </button>
        </footer>
      </article>
      <!-- Repeat card 4 times -->
    </div>
  </section>
</main>

<footer class="dashboard-footer">
  <!-- Console Log -->
  <section class="console-log">
    <h3 class="console-title">Activity Log</h3>
    <ul class="console-messages">
      <li>Task A completed ✅</li>
      <li>Task B in progress ⏳</li>
      <li>Task C failed ❌</li>
    </ul>
  </section>

  <!-- Progress Bars -->
  <section class="progress-section">
    <h3 class="progress-title">Task Progress</h3>
    <div class="progress-bar-group">
      <div class="progress-item">
        <span>Task A</span>
        <progress max="100" value="100"></progress>
      </div>
      <div class="progress-item">
        <span>Task B</span>
        <progress max="100" value="50"></progress>
      </div>
      <div class="progress-item">
        <span>Task C</span>
        <progress max="100" value="10"></progress>
      </div>
    </div>
  </section>
</footer>
```html
