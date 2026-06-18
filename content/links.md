+++
title = "Links"
date = "2026-06-17T17:08:16+08:00"
weight = 102
menu = "main"
+++

A small map of places I return to, things I build, and people I am glad to know online.

If you'd like to exchange links, feel free to reach out. Please submit [this](https://my.feishu.cn/share/base/form/shrcnBEu9mmLeio3pbLasFJCH2b)!

<section class="links-section">
  <div class="links-section-header">
    <p class="links-section-kicker">Regular reads</p>
  </div>
  <div class="links-grid links-grid--reads">
    <a href="https://www.ruanyifeng.com/blog/" target="_blank" rel="noopener" class="link-card">
      <img src="https://www.ruanyifeng.com/favicon.ico" class="link-avatar" loading="lazy">
      <div class="link-info">
        <span class="link-domain">www.ruanyifeng.com</span>
        <span class="link-name">阮一峰的网络日志</span>
      </div>
      <span class="link-arrow" aria-hidden="true">↗</span>
    </a>
    <a href="https://github.com/trending" target="_blank" rel="noopener" class="link-card">
      <img src="https://github.githubassets.com/favicons/favicon.svg" class="link-avatar" loading="lazy">
      <div class="link-info">
        <span class="link-domain">github.com/trending</span>
        <span class="link-name">GitHub Trending</span>
      </div>
      <span class="link-arrow" aria-hidden="true">↗</span>
    </a>
    <a href="https://github.com/trending" target="_blank" rel="noopener" class="link-card">
      <img src="https://www.v2ex.com/static/favicon.ico" class="link-avatar" loading="lazy">
      <div class="link-info">
        <span class="link-domain">www.v2ex.com</span>
        <span class="link-name">V2EX</span>
      </div>
      <span class="link-arrow" aria-hidden="true">↗</span>
    </a>
    <a href="https://space.bilibili.com/1815948385" target="_blank" rel="noopener" class="link-card">
      <img src="https://cdn.kmoon.fun/%E5%93%94%E5%93%A9%E5%93%94%E5%93%A9.png" class="link-avatar link-avatar--cover" loading="lazy">
      <div class="link-info">
        <span class="link-domain">space.bilibili.com</span>
        <span class="link-name">马克的技术工作坊</span>
      </div>
      <span class="link-arrow" aria-hidden="true">↗</span>
    </a>
  </div>
</section>

<!-- <section class="links-section">
  <div class="links-section-header">
    <p class="links-section-kicker">Things I build</p>
  </div>
  <div class="links-grid links-grid--works">
    <a href="/" class="link-card">
      <img src="https://cdn.kmoon.fun/profile.jpeg" class="link-avatar link-avatar--cover" loading="lazy">
      <div class="link-info">
        <span class="link-domain">this site</span>
        <span class="link-name">Kmoon's Blog</span>
      </div>
      <span class="link-arrow" aria-hidden="true">↗</span>
    </a>
    <a href="https://github.com/kmoonn/zhlgd-cli" class="link-card">
      <img src="https://github.githubassets.com/favicons/favicon.svg" class="link-avatar link-avatar--cover" loading="lazy">
      <div class="link-info">
        <span class="link-domain">zhlgd CLI</span>
        <span class="link-name">zhlgd-cli</span>
      </div>
      <span class="link-arrow" aria-hidden="true">↗</span>
    </a>
  </div>
</section> -->

<section class="links-section">
  <div class="links-section-header">
    <p class="links-section-kicker">Friends on the web</p>
  </div>
  <div class="links-grid links-grid--friends">
    <a href="https://blog.hazenix.top" target="_blank" rel="noopener" class="link-card">
      <img src="https://raw.githubusercontent.com/SomeBottle/somebottle/master/sticker/cut_the_potato.gif" class="link-avatar link-avatar--cover" loading="lazy">
      <div class="link-info">
        <span class="link-domain">bottle.moe</span>
        <span class="link-name">些瓶的小站</span>
        <span class="link-desc">相信的心就是你的魔法！</span>
      </div>
      <span class="link-arrow" aria-hidden="true">↗</span>
    </a>
    <a href="https://blog.hazenix.top" target="_blank" rel="noopener" class="link-card">
      <img src="https://blog.hazenix.top/assets/avatar-C7oPo7sC.jpg" class="link-avatar link-avatar--cover" loading="lazy">
      <div class="link-info">
        <span class="link-domain">blog.hazenix.top</span>
        <span class="link-name">Hazenix's Blog</span>
        <span class="link-desc">我会一直走，走到灯火通明。</span>
      </div>
      <span class="link-arrow" aria-hidden="true">↗</span>
    </a>
  </div>
</section>

<style>
.links-section {
  margin: 22px 0 26px;
}

.links-section-header {
  margin-bottom: 8px;
}

.links-section-kicker {
  margin: 0;
  font-family: var(--font-secondary), serif;
  font-size: 10px;
  letter-spacing: 0.1em;
  text-transform: uppercase;
  color: var(--text-color-tertiary);
}

.links-grid {
  --link-card-paper: #fbfcfe;
  --link-card-muted: #5b6675;
  --link-card-shadow: 0 8px 18px rgba(15, 23, 42, 0.05);
  --link-card-stripe-a: #5d8cc6;
  --link-card-stripe-b: #cb7e60;
  --link-card-stripe-c: #d8b45a;
  display: grid;
  gap: 12px;
  margin: 0;
}

.links-grid--reads,
.links-grid--works,
.links-grid--friends {
  grid-template-columns: repeat(2, minmax(0, 1fr));
}

.links-grid--works {
  --link-card-stripe-a: #7a8f68;
  --link-card-stripe-b: #5d8cc6;
  --link-card-stripe-c: #c6a45d;
}

.links-grid--friends {
  --link-card-stripe-a: #8b7ac8;
  --link-card-stripe-b: #d28b72;
  --link-card-stripe-c: #77a88d;
}

.link-card,
.link-card:visited,
.link-card:hover,
.link-card:active {
  position: relative;
  display: grid;
  grid-template-columns: 40px minmax(0, 1fr) auto;
  align-items: center;
  gap: 10px;
  height: 48px;
  padding: 10px 13px 10px;
  background: linear-gradient(180deg, var(--link-card-paper) 0%, var(--bg-color-secondary) 100%);
  border: 1px solid var(--border-color);
  border-radius: 14px;
  box-shadow: var(--link-card-shadow);
  text-decoration: none;
  color: inherit;
  overflow: hidden;
  transition: transform 0.18s ease, box-shadow 0.18s ease, border-color 0.18s ease, background-color 0.18s ease;
}

.link-card::before {
  content: "";
  position: absolute;
  inset: 0 0 auto;
  height: 4px;
  background: repeating-linear-gradient(
    90deg,
    var(--link-card-stripe-a) 0 18px,
    transparent 18px 24px,
    var(--link-card-stripe-b) 24px 42px,
    transparent 42px 48px,
    var(--link-card-stripe-c) 48px 66px,
    transparent 66px 72px
  );
}

.link-card:hover,
.link-card:focus-visible {
  transform: translateY(-2px);
  border-color: var(--link-color);
  box-shadow: 0 12px 22px rgba(15, 23, 42, 0.09);
  text-decoration: none;
}

.link-card:focus-visible {
  outline: 2px solid var(--link-color);
  outline-offset: 3px;
}

.link-avatar {
  width: 40px;
  height: 40px;
  margin: 0;
  padding: 5px;
  border-radius: 12px;
  box-sizing: border-box;
  border: 1px solid var(--border-color);
  background: linear-gradient(180deg, rgba(255, 255, 255, 0.92) 0%, rgba(244, 247, 251, 0.92) 100%);
  overflow: hidden;
  object-fit: contain;
}

.link-avatar--cover {
  padding: 0;
  object-fit: cover;
}

.link-card:hover .link-avatar,
.link-card:focus-visible .link-avatar {
  transform: translateY(-1px);
  border-color: var(--link-color);
}

.link-info {
  display: grid;
  gap: 2px;
  min-width: 0;
}

.link-domain {
  font-family: var(--font-secondary), serif;
  font-size: 10px;
  letter-spacing: 0.08em;
  text-transform: uppercase;
  color: var(--link-card-muted);
}

.link-name {
  font-size: 14px;
  font-weight: 700;
  color: var(--text-color-primary);
  line-height: 1.3;
}

.link-desc {
  font-size: 12px;
  color: var(--text-color-secondary);
  line-height: 1.45;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

.link-arrow {
  align-self: center;
  font-size: 16px;
  line-height: 1;
  color: var(--text-color-tertiary);
  transition: transform 0.18s ease, color 0.18s ease;
}

.link-card:hover .link-arrow,
.link-card:focus-visible .link-arrow {
  color: var(--link-color);
  transform: translate(2px, -2px);
}

[data-theme="dark"] .links-grid {
  --link-card-paper: #151a22;
  --link-card-muted: #9ba8b8;
  --link-card-shadow: 0 12px 24px rgba(0, 0, 0, 0.24);
  --link-card-stripe-a: #78aef3;
  --link-card-stripe-b: #f09a78;
  --link-card-stripe-c: #e0c36d;
}

[data-theme="dark"] .links-grid--works {
  --link-card-stripe-a: #9fbc88;
  --link-card-stripe-b: #78aef3;
  --link-card-stripe-c: #e0c36d;
}

[data-theme="dark"] .links-grid--friends {
  --link-card-stripe-a: #b7a0ff;
  --link-card-stripe-b: #f3a184;
  --link-card-stripe-c: #86c7a5;
}

[data-theme="dark"] .link-card:hover,
[data-theme="dark"] .link-card:focus-visible {
  box-shadow: 0 16px 28px rgba(0, 0, 0, 0.32);
}

[data-theme="dark"] .link-avatar {
  background: linear-gradient(180deg, rgba(28, 36, 48, 0.92) 0%, rgba(22, 28, 38, 0.92) 100%);
}

@media (prefers-color-scheme: dark) {
  :root:not([data-theme="light"]) .links-grid {
    --link-card-paper: #151a22;
    --link-card-muted: #9ba8b8;
    --link-card-shadow: 0 12px 24px rgba(0, 0, 0, 0.24);
    --link-card-stripe-a: #78aef3;
    --link-card-stripe-b: #f09a78;
    --link-card-stripe-c: #e0c36d;
  }

  :root:not([data-theme="light"]) .links-grid--works {
    --link-card-stripe-a: #9fbc88;
    --link-card-stripe-b: #78aef3;
    --link-card-stripe-c: #e0c36d;
  }

  :root:not([data-theme="light"]) .links-grid--friends {
    --link-card-stripe-a: #b7a0ff;
    --link-card-stripe-b: #f3a184;
    --link-card-stripe-c: #86c7a5;
  }

  :root:not([data-theme="light"]) .link-card:hover,
  :root:not([data-theme="light"]) .link-card:focus-visible {
    box-shadow: 0 16px 28px rgba(0, 0, 0, 0.32);
  }

  :root:not([data-theme="light"]) .link-avatar {
    background: linear-gradient(180deg, rgba(28, 36, 48, 0.92) 0%, rgba(22, 28, 38, 0.92) 100%);
  }
}

@media (max-width: 640px) {
  .links-section {
    margin: 20px 0 24px;
  }

  .links-grid,
  .links-grid--reads,
  .links-grid--works,
  .links-grid--friends {
    grid-template-columns: 1fr;
    gap: 10px;
  }

  .link-card,
  .link-card:visited,
  .link-card:hover,
  .link-card:active {
    grid-template-columns: 36px minmax(0, 1fr);
    min-height: 32px;
    padding: 11px 12px 9px;
  }

  .link-avatar {
    width: 36px;
    height: 36px;
    padding: 4px;
    border-radius: 11px;
  }

  .link-arrow {
    display: none;
  }
}

@media (prefers-reduced-motion: reduce) {
  .link-card,
  .link-avatar,
  .link-arrow {
    transition: none;
  }

  .link-card:hover,
  .link-card:focus-visible,
  .link-card:hover .link-avatar,
  .link-card:focus-visible .link-avatar,
  .link-card:hover .link-arrow,
  .link-card:focus-visible .link-arrow {
    transform: none;
  }
}
</style>
