<script>
  import { onMount, onDestroy } from 'svelte';
  import { supabase } from './lib/supabaseClient';
  import Post from './lib/Post.svelte';

  let posts = [];
  let newNote = '';
  let selectedFile = null;
  let fileType = null;
  let uploading = false;
  let scrollY = 0;

  async function fetchPosts() {
    const { data, error } = await supabase
      .from('posts')
      .select('*')
      .order('created_at', { ascending: false });
    if (!error) posts = data;
  }

  let subscription;
  onMount(() => {
    fetchPosts();
    subscription = supabase
      .channel('public:posts')
      .on('postgres_changes', { event: 'INSERT', schema: 'public', table: 'posts' }, (payload) => {
        posts = [payload.new, ...posts];
      })
      .subscribe();

    window.addEventListener('scroll', () => { scrollY = window.scrollY; });
  });

  onDestroy(() => {
    supabase.removeChannel(subscription);
    window.removeEventListener('scroll', () => {});
  });

  function handleFileChange(e) {
    const file = e.target.files[0];
    if (!file) return;
    selectedFile = file;
    if (file.type.startsWith('image/')) fileType = 'image';
    else if (file.type.startsWith('video/')) fileType = 'video';
    else if (file.type.startsWith('audio/')) fileType = 'audio';
  }

  async function submitPost() {
    uploading = true;
    try {
      let mediaUrl = null;
      let text = newNote.trim();
      let type = 'note';

      if (selectedFile) {
        const fileName = `${Date.now()}_${selectedFile.name.replace(/\s+/g, '_')}`;
        const { data, error } = await supabase.storage
          .from('mecho-media')
          .upload(`public/${fileName}`, selectedFile, { cacheControl: '3600' });

        if (error) throw error;
        const { data: urlData } = supabase.storage.from('mecho-media').getPublicUrl(data.path);
        mediaUrl = urlData.publicUrl;
        type = fileType;
      } else if (!text) {
        uploading = false;
        return;
      }

      const { error } = await supabase.from('posts').insert([{
        type,
        text: text || null,
        media_url: mediaUrl,
        created_at: new Date().toISOString()
      }]);

      if (error) throw error;
      newNote = '';
      selectedFile = null;
      fileType = null;
    } catch (err) {
      console.error('Error:', err);
      alert('No se pudo enviar. Intenta de nuevo.');
    } finally {
      uploading = false;
    }
  }
</script>

<main>
  <header>
    <div class="logo">mecho 🌿</div>
    <p class="subtitle">un rincón suave para nosotras</p>
  </header>

  <!-- Burbuja con Mew (reemplaza emojis por tus imágenes pixel art) -->
  <div class="mew-bubble" style="--parallax: {scrollY * 0.08}px;">
    <div class="bubble">
      <span class="mew">🤍🫧</span>
    </div>
  </div>

  <section class="upload-card">
    <textarea bind:value={newNote} placeholder="Escribe un mensajito bonito..." rows="3"></textarea>
    <input type="file" accept="image/*,video/*,audio/*" on:change={handleFileChange} id="file-input" />
    <label for="file-input" class="file-label">📎 Adjuntar foto, video o audio</label>
    <button on:click={submitPost} disabled={uploading || (!newNote.trim() && !selectedFile)} class="submit-btn">
      {uploading ? 'Soltando en el muro...' : 'Enviar ✨'}
    </button>
  </section>

  <section class="feed">
    {#each posts as post (post.id)}
      <Post {post} />
    {/each}
  </section>

  <img src="/mecho.png" alt="Mecha" class="mascot" />
</main>

<style>
  @import url('https://fonts.googleapis.com/css2?family=Nunito:wght@400;600;700&family=VT323&display=swap');

  :root {
    --bg: #F5F9F6;
    --card: #FFFFFF;
    --green: #8B9A7C;
    --blue: #A8C3D6;
    --pink: #F4C2C2;
    --sand: #E8D5B7;
    --text: #4A4A4A;
    --text-light: #7A7A7A;
  }

  * { box-sizing: border-box; margin: 0; padding: 0; }
  body {
    background: var(--bg);
    color: var(--text);
    font-family: 'Nunito', sans-serif;
    scroll-behavior: smooth;
    min-height: 100vh;
  }

  main {
    max-width: 640px;
    margin: 0 auto;
    padding: 24px 16px 100px;
    position: relative;
  }

  header { text-align: center; margin-bottom: 28px; }
  .logo {
    font-family: 'VT323', monospace;
    font-size: 2.6rem;
    color: var(--green);
    letter-spacing: 3px;
  }
  .subtitle { color: var(--text-light); font-size: 1rem; margin-top: 4px; }

  .upload-card {
    background: var(--card);
    border-radius: 30px;
    padding: 20px;
    box-shadow: 0 8px 24px rgba(139, 154, 124, 0.14);
    display: flex;
    flex-direction: column;
    gap: 14px;
    margin-bottom: 36px;
    border: 2px solid var(--sand);
  }

  textarea {
    width: 100%;
    border: none;
    background: var(--sand);
    border-radius: 18px;
    padding: 14px;
    font-family: 'Nunito', sans-serif;
    font-size: 1.05rem;
    resize: vertical;
    color: var(--text);
  }
  textarea:focus { outline: 2px solid var(--pink); }

  input[type="file"] { display: none; }
  .file-label {
    background: var(--blue);
    color: white;
    padding: 11px 16px;
    border-radius: 22px;
    text-align: center;
    cursor: pointer;
    font-weight: 600;
    transition: background 0.2s;
  }
  .file-label:hover { background: #8FB5CC; }

  .submit-btn {
    background: var(--green);
    color: white;
    border: none;
    padding: 15px;
    border-radius: 26px;
    font-family: 'Nunito', sans-serif;
    font-weight: 700;
    font-size: 1.05rem;
    cursor: pointer;
    transition: transform 0.15s, opacity 0.2s;
  }
  .submit-btn:hover:not(:disabled) { transform: scale(1.02); }
  .submit-btn:disabled { opacity: 0.6; cursor: not-allowed; }

  .feed { display: flex; flex-direction: column; gap: 16px; }

  .mascot {
    position: fixed;
    bottom: 24px;
    right: 18px;
    width: 64px;
    height: auto;
    animation: floatMascot 4s ease-in-out infinite;
    z-index: 20;
    filter: drop-shadow(0 4px 8px rgba(0,0,0,0.1));
  }
  @keyframes floatMascot {
    0%, 100% { transform: translateY(0) rotate(-2deg); }
    50% { transform: translateY(-12px) rotate(2deg); }
  }

  .mew-bubble {
    position: fixed;
    top: 12%;
    left: 12px;
    width: 90px;
    height: 110px;
    z-index: 5;
    pointer-events: none;
    transform: translateY(var(--parallax));
    transition: transform 0.1s linear;
  }
  .bubble {
    background: rgba(255, 255, 255, 0.65);
    border-radius: 50%;
    width: 100%;
    height: 100%;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 2.2rem;
    box-shadow: 0 10px 30px rgba(168, 195, 214, 0.25);
    animation: bubbleDrift 14s ease-in-out infinite alternate;
  }
  @keyframes bubbleDrift {
    0% { transform: translateY(0) rotate(-6deg); }
    100% { transform: translateY(35px) rotate(6deg); }
  }
</style>