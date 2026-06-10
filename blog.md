---
layout: custom
title: "Research Blog - Changjae Lee"
permalink: /blog/
---

<div class="blog-archive-wrapper">
    <header class="blog-archive-header">
        <h1 class="section-title">Research Blog</h1>
        <p class="blog-archive-desc">Notes, guides, experiments, and observations on agentic AI, databases, and human-computer systems.</p>
    </header>

    <!-- Category Filters -->
    <div class="blog-filters">
        <button class="filter-btn active" data-filter="all">All Posts</button>
        <button class="filter-btn" data-filter="tutorials">Tutorials</button>
        <button class="filter-btn" data-filter="debugging">Debugging</button>
        <button class="filter-btn" data-filter="experiments">Experiments</button>
        <button class="filter-btn" data-filter="research-notes">Research Notes</button>
    </div>

    <!-- Posts Grid -->
    <div class="blog-posts-grid">
        {% if site.posts.size > 0 %}
            {% for post in site.posts %}
                <article class="blog-post-card" data-category="{{ post.category | downcase }}">
                    <div class="blog-card-meta">
                        <span class="blog-card-date">{{ post.date | date: "%B %d, %Y" }}</span>
                        <span class="blog-category-badge category-{{ post.category | downcase }}">{{ post.category }}</span>
                    </div>
                    <h2 class="blog-post-title"><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
                    <p class="blog-post-excerpt">{{ post.excerpt | strip_html | truncatewords: 30 }}</p>
                    <div class="blog-card-footer">
                        <a href="{{ post.url | relative_url }}" class="read-more-btn">
                            Read Post
                            <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7" />
                            </svg>
                        </a>
                    </div>
                </article>
            {% endfor %}
        {% else %}
            <div class="blog-empty-state">
                <div class="empty-icon">📝</div>
                <h3>No posts yet</h3>
                <p>Blog posts will appear here once published. Stay tuned!</p>
            </div>
        {% endif %}
    </div>
</div>

<script>
    document.addEventListener('DOMContentLoaded', () => {
        const filterButtons = document.querySelectorAll('.filter-btn');
        const postCards = document.querySelectorAll('.blog-post-card');
        const postsGrid = document.querySelector('.blog-posts-grid');
        
        // Add dynamic empty-state element if there are posts but none match the current filter
        let categoryEmptyMessage = document.getElementById('category-empty-message');
        if (!categoryEmptyMessage && postCards.length > 0) {
            categoryEmptyMessage = document.createElement('div');
            categoryEmptyMessage.id = 'category-empty-message';
            categoryEmptyMessage.className = 'blog-empty-state';
            categoryEmptyMessage.style.display = 'none';
            categoryEmptyMessage.innerHTML = `
                <div class="empty-icon">🔍</div>
                <h3>No posts in this category</h3>
                <p>There are no posts matching this category yet. Check out other categories!</p>
            `;
            postsGrid.appendChild(categoryEmptyMessage);
        }

        filterButtons.forEach(button => {
            button.addEventListener('click', () => {
                // Remove active class from all buttons
                filterButtons.forEach(btn => btn.classList.remove('active'));
                // Add active class to clicked button
                button.classList.add('active');

                const filterValue = button.getAttribute('data-filter');
                let visibleCount = 0;

                postCards.forEach(card => {
                    const cardCategory = card.getAttribute('data-category');
                    if (filterValue === 'all' || cardCategory === filterValue) {
                        card.style.display = 'flex';
                        visibleCount++;
                    } else {
                        card.style.display = 'none';
                    }
                });

                if (categoryEmptyMessage) {
                    if (visibleCount === 0) {
                        categoryEmptyMessage.style.display = 'flex';
                    } else {
                        categoryEmptyMessage.style.display = 'none';
                    }
                }
            });
        });
    });
</script>
