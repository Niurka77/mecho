<script>
  export let post;
  export let onLike = () => {};
  
  let liked = false;
  let localLikes = post.likes || 0;
  
  function formatTime(dateString) {
    const d = new Date(dateString);
    const now = new Date();
    const diffMs = now - d;
    const diffMins = Math.floor(diffMs / 60000);
    const diffHours = Math.floor(diffMs / 3600000);
    const diffDays = Math.floor(diffMs / 86400000);
    
    if (diffMins < 1) return 'ahorita 💫';
    if (diffMins < 60) return `hace ${diffMins} ${diffMins === 1 ? 'min' : 'mins'} 🌸`;
    if (diffHours < 24) return `hace ${diffHours} ${diffHours === 1 ? 'hora' : 'horas'} ✨`;
    if (diffDays < 7) return `hace ${diffDays} ${diffDays === 1 ? 'día' : 'días'} 🫧`;
    
    return d.toLocaleString('es-ES', { 
      day: 'numeric', month: 'short', hour: '2-digit', minute: '2-digit'
    });
  }
  
  function handleLike() {
    liked = !liked;
    localLikes = localLikes + (liked ? 1 : -1);
    onLike(post.id, liked);
  }
</script>

<article class="post-card" style="background-color: {post.color || '#FFFFFF'}">
  <!-- Badge de mood -->
  {#if post.mood_emoji}
    <div class="mood-badge">
      <span class="mood-emoji">{post.mood_emoji}</span>
      <span class="mood-label">{post.mood}</span>
    </div>
  {/if}
  
  {#if post.text}
    <p class="post-text">{post.text}</p>
  {/if}
  
  {#if post.media_url}
    <div class="media-wrapper">
      {#if post.type === 'image'}
        <img src={post.media_url} alt="Recuerdo compartido" loading="lazy" class="post-media" />
      {:else if post.type === 'video'}
        <video controls src={post.media_url} class="post-media" preload="metadata"></video>
      {:else if post.type === 'audio'}
        <audio controls src={post.media_url} class="post-audio"></audio>
      {/if}
    </div>
  {/if}

  <div class="post-footer">
    <time class="post-time">{formatTime(post.created_at)}</time>
    <button class="like-btn {liked ? 'liked' : ''}" on:click={handleLike}>
      {liked ? '💖' : '🩶'} 
      <span class="like-count">{localLikes}</span>
    </button>
  </div>
</article>

<style>
  .post-card {
    background: #FFFFFF;
    border-radius: 28px 32px 24px 36px;
    padding: 20px 22px;
    box-shadow: 0 8px 24px rgba(139, 154, 124, 0.1);
    display: flex;
    flex-direction: column;
    gap: 14px;
    transition: all 0.35s cubic-bezier(0.34, 1.2, 0.64, 1);
    border: 2px solid rgba(232, 213, 183, 0.4);
    position: relative;
    overflow: hidden;
  }
  
  .post-card::before {
    content: '';
    position: absolute;
    top: 0; left: 0; right: 0; bottom: 0;
    background: radial-gradient(circle at 30% 20%, rgba(255,255,255,0.4) 0%, transparent 60%);
    pointer-events: none;
  }
  
  .post-card:hover { 
    transform: translateY(-6px) scale(1.01); 
    box-shadow: 0 18px 40px rgba(139, 154, 124, 0.2); 
    border-color: rgba(232, 213, 183, 0.7);
  }
  
  /* === BADGE DE MOOD === */
  .mood-badge {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    background: rgba(255,255,255,0.7);
    padding: 4px 12px;
    border-radius: 20px;
    font-size: 0.8rem;
    font-weight: 600;
    color: #5A5A5A;
    align-self: flex-start;
    backdrop-filter: blur(4px);
    border: 1px solid rgba(200,220,255,0.4);
  }
  
  .mood-emoji { font-size: 1.1rem; }
  .mood-label { opacity: 0.9; }
  
  .post-text {
    font-size: 1.05rem;
    line-height: 1.65;
    color: #4A4A4A;
    white-space: pre-wrap;
    word-wrap: break-word;
    position: relative;
    z-index: 1;
    font-weight: 500;
  }
  
  .media-wrapper {
    position: relative;
    border-radius: 22px;
    overflow: hidden;
  }
  
  .post-media {
    width: 100%;
    border-radius: 22px;
    object-fit: cover;
    max-height: 420px;
    transition: transform 0.4s ease;
  }
  
  .media-wrapper:hover .post-media {
    transform: scale(1.02);
  }
  
  .post-audio {
    width: 100%;
    border-radius: 30px;
    background: rgba(249, 247, 243, 0.9);
  }
  
  .post-footer {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-top: 6px;
  }
  
  .post-time {
    font-size: 0.7rem;
    color: #A8B8A0;
    font-family: 'VT323', monospace;
    letter-spacing: 0.8px;
    opacity: 0.85;
    font-weight: 500;
  }
  
  .like-btn {
    background: rgba(232, 213, 183, 0.3);
    border: none;
    font-size: 1rem;
    cursor: pointer;
    display: flex;
    align-items: center;
    gap: 6px;
    padding: 6px 14px;
    border-radius: 40px;
    transition: all 0.25s ease;
    color: #A8B8A0;
    font-weight: 600;
  }
  
  .like-btn:hover {
    transform: scale(1.08);
    background: rgba(232, 213, 183, 0.5);
  }
  
  .like-btn:active { transform: scale(0.95); }
  
  .like-btn.liked {
    background: rgba(244, 194, 194, 0.35);
    color: #F4C2C2;
  }
  
  .like-count {
    font-size: 0.75rem;
    font-family: monospace;
    font-weight: 600;
  }
  
  @keyframes gentlePop {
    0% { transform: scale(1); }
    50% { transform: scale(1.05); background: rgba(244, 194, 194, 0.5); }
    100% { transform: scale(1); }
  }
  
  .like-btn.liked { animation: gentlePop 0.3s ease; }
</style>