<script>
  export let post;
  export let onLike = () => {};
  export let darkMode = true;
  
  let liked = false;
  let localLikes = post.likes || 0;
  
  function formatTime(dateString) {
    const d = new Date(dateString);
    const now = new Date();
    const diffMs = now - d;
    const diffMins = Math.floor(diffMs / 60000);
    const diffHours = Math.floor(diffMs / 3600000);
    const diffDays = Math.floor(diffMs / 86400000);
    
    if (diffMins < 1) return 'now ✦';
    if (diffMins < 60) return `${diffMins}m ago ✿`;
    if (diffHours < 24) return `${diffHours}h ago ★`;
    if (diffDays < 7) return `${diffDays}d ago ❀`;
    
    return d.toLocaleString('en-US', { 
      month: 'short', day: 'numeric', hour: '2-digit', minute: '2-digit'
    }).replace(',', '');
  }
  
  function handleLike() {
    liked = !liked;
    localLikes = localLikes + (liked ? 1 : -1);
    onLike(post.id, liked);
  }
</script>

<article class="post-card" style="--mood-border: {post.mood_border || '#A8B8A0'}">
  
  <!-- Badge de Mood estilo retro -->
  {#if post.mood_emoji}
    <div class="mood-badge">
      <span class="mood-emoji">{post.mood_emoji}</span>
      <span class="mood-label">{post.mood?.toLowerCase()}</span>
    </div>
  {/if}
  
  <!-- Texto del post -->
  {#if post.text}
    <p class="post-text">{post.text}</p>
  {/if}
  
  <!-- Media -->
  {#if post.media_url}
    <div class="media-frame">
      {#if post.type === 'image'}
        <img src={post.media_url} alt="entry" loading="lazy" class="post-media" />
      {:else if post.type === 'video'}
        <video controls src={post.media_url} class="post-media" preload="metadata"></video>
      {:else if post.type === 'audio'}
        <div class="audio-box">
          <span class="audio-icon">♪</span>
          <audio controls src={post.media_url} class="post-audio"></audio>
        </div>
      {/if}
    </div>
  {/if}

  <!-- Footer con likes y fecha -->
  <div class="post-footer">
    <time class="post-time">{formatTime(post.created_at)}</time>
    <button class="like-btn {liked ? 'liked' : ''}" on:click={handleLike} type="button">
      <span class="like-icon">{liked ? '♥' : '♡'}</span>
      <span class="like-count">{localLikes}</span>
    </button>
  </div>
  
  <!-- Borde decorativo inferior -->
  <div class="post-decoration"></div>
</article>

<style>
  .post-card {
    background: var(--bg-card);
    border: var(--border-width) solid var(--border-default);
    outline: var(--border-width) solid var(--mood-border);
    outline-offset: -2px;
    border-radius: var(--radius);
    padding: 16px;
    margin-bottom: 16px;
    position: relative;
    transition: all 0.1s steps(2);
  }
  
  .post-card:hover {
    outline-color: var(--border-rose);
    transform: translateX(2px);
  }
  
  /* Badge de Mood */
  .mood-badge {
    display: inline-flex;
    align-items: center;
    gap: 4px;
    background: var(--bg-header);
    border: var(--border-width) solid var(--mood-border);
    padding: 3px 10px;
    border-radius: var(--radius);
    font-size: 0.85rem;
    margin-bottom: 10px;
    color: var(--text-muted);
  }
  
  .mood-emoji { font-size: 1rem; }
  .mood-label { text-transform: lowercase; }
  
  /* Texto */
  .post-text {
    font-size: 1.1rem;
    line-height: 1.6;
    color: var(--text-main);
    white-space: pre-wrap;
    word-wrap: break-word;
    margin-bottom: 12px;
  }
  
  /* Frame de media */
  .media-frame {
    border: var(--border-width) dashed var(--border-default);
    border-radius: var(--radius);
    overflow: hidden;
    margin-bottom: 12px;
  }
  
  .post-media {
    width: 100%;
    display: block;
    image-rendering: pixelated;
    max-height: 360px;
    object-fit: cover;
  }
  
  .audio-box {
    padding: 10px;
    display: flex;
    align-items: center;
    gap: 8px;
  }
  
  .audio-icon {
    font-size: 1.2rem;
    color: var(--border-rose);
  }
  
  .post-audio {
    width: 100%;
    filter: invert(var(--audio-filter, 0));
  }
  
  /* Footer */
  .post-footer {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding-top: 10px;
    border-top: var(--border-dotted) var(--border-default);
  }
  
  .post-time {
    font-size: 0.85rem;
    color: var(--text-muted);
    letter-spacing: 0.5px;
  }
  
  /* Botón Like estilo retro */
  .like-btn {
    background: transparent;
    border: var(--border-width) solid var(--border-default);
    color: var(--text-muted);
    padding: 4px 12px;
    border-radius: var(--radius);
    font-family: 'VT323', monospace;
    font-size: 0.9rem;
    cursor: pointer;
    display: flex;
    align-items: center;
    gap: 4px;
    transition: all 0.1s steps(2);
  }
  
  .like-btn:hover {
    border-color: var(--border-rose);
    color: var(--border-rose);
  }
  
  .like-btn.liked {
    border-color: var(--border-rose);
    color: var(--border-rose);
    background: rgba(244, 194, 194, 0.1);
  }
  
  .like-icon { font-size: 1.1rem; }
  .like-count { font-size: 0.9rem; }
  
  /* Decoración inferior pixelada */
  .post-decoration {
    position: absolute;
    bottom: -2px;
    left: 8px;
    right: 8px;
    height: 2px;
    background: repeating-linear-gradient(
      90deg,
      var(--mood-border) 0px,
      var(--mood-border) 4px,
      transparent 4px,
      transparent 8px
    );
    pointer-events: none;
  }
  
  /* Animación pixel al dar like */
  @keyframes pixelHeart {
    0% { transform: scale(1); }
    50% { transform: scale(1.3); }
    100% { transform: scale(1); }
  }
  
  .like-btn.liked .like-icon {
    animation: pixelHeart 0.2s steps(4);
  }
</style>