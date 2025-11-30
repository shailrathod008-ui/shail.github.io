---
layout: page
title: "LeetCode Problems Index"
permalink: /leetcode/
description: "All my LeetCode solutions with Hinglish explanations, difficulty tags, and topics."
---

> Saare LeetCode posts ek jagah 😎 – difficulty badge + tags ke saath.

<div class="leetcode-filters">
  <button class="lc-filter-btn" data-diff="all">All</button>
  <button class="lc-filter-btn" data-diff="easy">Easy</button>
  <button class="lc-filter-btn" data-diff="medium">Medium</button>
  <button class="lc-filter-btn" data-diff="hard">Hard</button>
</div>

<hr>

<div class="leetcode-list">
  {% assign lc_posts = site.posts | where_exp: "post", "post.tags contains 'leetcode'" %}
  {% assign lc_posts = lc_posts | sort: "date" | reverse %}

  {% for post in lc_posts %}
    {% assign diff_class = 'unknown' %}
    {% assign diff_label = 'Unspecified' %}
    {% if post.tags contains 'easy' %}
      {% assign diff_class = 'easy' %}
      {% assign diff_label = 'Easy' %}
    {% elsif post.tags contains 'medium' %}
      {% assign diff_class = 'medium' %}
      {% assign diff_label = 'Medium' %}
    {% elsif post.tags contains 'hard' %}
      {% assign diff_class = 'hard' %}
      {% assign diff_label = 'Hard' %}
    {% endif %}

    <article class="lc-item" data-diff="{{ diff_class }}">
      <h3 class="lc-title">
        <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
      </h3>

      <div class="lc-meta">
        <span class="lc-date">{{ post.date | date: "%Y-%m-%d" }}</span>
        <span class="lc-diff-badge lc-diff-{{ diff_class }}">{{ diff_label }}</span>
      </div>

      {% if post.description %}
        <p class="lc-desc">{{ post.description }}</p>
      {% endif %}

      <div class="lc-tags">
        {% for tag in post.tags %}
          <span class="lc-tag">#{{ tag }}</span>
        {% endfor %}
      </div>
    </article>

    <hr>
  {% endfor %}
</div>

<script>
  document.addEventListener("DOMContentLoaded", function () {
    const buttons = document.querySelectorAll(".lc-filter-btn");
    const items = document.querySelectorAll(".lc-item");

    buttons.forEach(btn => {
      btn.addEventListener("click", () => {
        const diff = btn.getAttribute("data-diff");

        buttons.forEach(b => b.classList.remove("active"));
        btn.classList.add("active");

        items.forEach(item => {
          if (diff === "all") {
            item.style.display = "";
          } else {
            const itemDiff = item.getAttribute("data-diff");
            item.style.display = (itemDiff === diff) ? "" : "none";
          }
        });
      });
    });
  });
</script>
