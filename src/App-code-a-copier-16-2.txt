import React, { useState, useEffect } from "react";

/* =============================================================================
 * STAR DAV — mini-site e-commerce
 * =============================================================================
 * Un seul vendeur (STAR DAV) : catalogue organisé par catégorie, commande
 * directe sur WhatsApp par produit, espace admin protégé par mot de passe
 * pour gérer les produits et le logo. Même architecture que BK Traffic /
 * ANTYshop (React + Vite + Supabase + Vercel), adaptée à une seule boutique.
 * ============================================================================= */

const PHONE_DISPLAY = "+226 05 34 28 27";
const WHATSAPP_NUMBER = "22605342827";
const TIKTOK = "@stardav555";
const TIKTOK_URL = "https://www.tiktok.com/@stardav555";
const LOCATION = "Ouagadougou, Burkina Faso";
const ADMIN_PASSWORD = "stardav2026"; // à changer une fois le site en ligne

const waLink = (message) => `https://wa.me/${WHATSAPP_NUMBER}?text=${encodeURIComponent(message)}`;

const CATEGORIES = [
  { key: "vetements", label: "Vêtements", emoji: "👕" },
  { key: "chaussures", label: "Chaussures", emoji: "👟" },
  { key: "maillots", label: "Maillots", emoji: "⚽" },
  { key: "stanley", label: "Stanley & accessoires", emoji: "🥤" },
];

/* Catalogue de départ — inséré automatiquement dans Supabase au tout premier
   chargement si la table est vide (voir seedProducts). Une fois de vraies
   données ajoutées depuis l'admin, ce tableau ne sert plus. */
const DEFAULT_PRODUCTS = [
  { name: "Crocs Classique", category: "chaussures", price: 5000, image: "https://picsum.photos/seed/stardavcrocs/700/700", description: "Sabots confortables, parfaits pour un usage quotidien. Plusieurs coloris disponibles selon le stock.", sizes: "38,39,40,41,42,43", available: true },
  { name: "Sandale", category: "chaussures", price: 4000, image: "https://picsum.photos/seed/stardavsandale/700/700", description: "Sandale tendance, confortable et légère, idéale pour la saison chaude.", sizes: "37,38,39,40,41", available: true },
  { name: "Maillot", category: "maillots", price: 4000, image: "https://picsum.photos/seed/stardavmaillot/700/700", description: "Maillot de sport, coupe confortable. Disponible pour plusieurs équipes/modèles selon arrivage.", sizes: "S,M,L,XL", available: true },
  { name: "Stanley", category: "stanley", price: 6000, image: "https://picsum.photos/seed/stardavstanley/700/700", description: "Gourde isotherme Stanley, garde vos boissons chaudes ou froides longtemps.", sizes: "", available: true },
];

/* ---------- SUPABASE ---------- */
const SUPABASE_URL = "https://tbxriylzbqwwwqafielj.supabase.co";
const SUPABASE_ANON_KEY = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InRieHJpeWx6YnF3d3dxYWZpZWxqIiwicm9sZSI6ImFub24iLCJpYXQiOjE3ODY2MjI2ODMsImV4cCI6MjEwMjE5ODY4M30.ptlOGSTIeH2iglF_ltDB6_DVhNtlwDKPVcyYqeJNXKQ";

const sbHeaders = (extra = {}) => ({
  apikey: SUPABASE_ANON_KEY,
  Authorization: `Bearer ${SUPABASE_ANON_KEY}`,
  "Content-Type": "application/json",
  ...extra,
});

async function fetchProducts() {
  try {
    const res = await fetch(`${SUPABASE_URL}/rest/v1/products?select=*&order=created_at.asc`, { headers: sbHeaders() });
    if (!res.ok) throw new Error("fetch failed");
    return await res.json();
  } catch (e) {
    console.error("Erreur chargement produits", e);
    return null;
  }
}
async function seedProducts(products) {
  try {
    await fetch(`${SUPABASE_URL}/rest/v1/products`, {
      method: "POST",
      headers: sbHeaders({ Prefer: "resolution=merge-duplicates,return=minimal" }),
      body: JSON.stringify(products),
    });
  } catch (e) {
    console.error("Erreur initialisation produits", e);
  }
}
async function insertProductRow(product) {
  try {
    const res = await fetch(`${SUPABASE_URL}/rest/v1/products`, {
      method: "POST",
      headers: sbHeaders({ Prefer: "return=representation" }),
      body: JSON.stringify([product]),
    });
    if (!res.ok) return null;
    const data = await res.json();
    return data[0];
  } catch (e) {
    console.error("Erreur ajout produit", e);
    return null;
  }
}
async function updateProductRow(id, patch) {
  try {
    await fetch(`${SUPABASE_URL}/rest/v1/products?id=eq.${encodeURIComponent(id)}`, {
      method: "PATCH",
      headers: sbHeaders({ Prefer: "return=minimal" }),
      body: JSON.stringify(patch),
    });
  } catch (e) {
    console.error("Erreur modification produit", e);
  }
}
async function deleteProductRow(id) {
  try {
    await fetch(`${SUPABASE_URL}/rest/v1/products?id=eq.${encodeURIComponent(id)}`, {
      method: "DELETE",
      headers: sbHeaders(),
    });
  } catch (e) {
    console.error("Erreur suppression produit", e);
  }
}
async function uploadImage(file, folder = "products") {
  try {
    const ext = (file.name.split(".").pop() || "jpg").toLowerCase();
    const path = `${folder}/${Date.now()}-${Math.random().toString(36).slice(2, 8)}.${ext}`;
    const res = await fetch(`${SUPABASE_URL}/storage/v1/object/stardav-images/${path}`, {
      method: "POST",
      headers: {
        apikey: SUPABASE_ANON_KEY,
        Authorization: `Bearer ${SUPABASE_ANON_KEY}`,
        "Content-Type": file.type || "application/octet-stream",
      },
      body: file,
    });
    if (!res.ok) throw new Error("upload failed");
    return `${SUPABASE_URL}/storage/v1/object/public/stardav-images/${path}`;
  } catch (e) {
    console.error("Erreur upload image", e);
    return null;
  }
}
async function fetchSiteSetting(key) {
  try {
    const res = await fetch(`${SUPABASE_URL}/rest/v1/site_settings?select=*&key=eq.${encodeURIComponent(key)}`, { headers: sbHeaders() });
    if (!res.ok) throw new Error("fetch setting failed");
    const data = await res.json();
    return data[0]?.value || null;
  } catch (e) {
    console.error("Erreur chargement réglage", e);
    return null;
  }
}
async function setSiteSetting(key, value) {
  try {
    await fetch(`${SUPABASE_URL}/rest/v1/site_settings`, {
      method: "POST",
      headers: sbHeaders({ Prefer: "resolution=merge-duplicates,return=minimal" }),
      body: JSON.stringify([{ key, value }]),
    });
  } catch (e) {
    console.error("Erreur sauvegarde réglage", e);
  }
}

const fmt = (n) => new Intl.NumberFormat("fr-FR").format(n) + " FCFA";
const sizesToArray = (s) => (s || "").split(",").map((x) => x.trim()).filter(Boolean);

/* ---------- ICÔNES ---------- */
const Icon = {
  chat: (p) => (<svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" strokeWidth="1.8" {...p}><path d="M4 12a8 8 0 1 1 3.2 6.4L4 20l1.3-3.6A7.96 7.96 0 0 1 4 12Z" strokeLinecap="round" strokeLinejoin="round" /></svg>),
  search: (p) => (<svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" strokeWidth="1.8" {...p}><circle cx="11" cy="11" r="6.5" /><path d="m20 20-4.3-4.3" strokeLinecap="round" /></svg>),
  back: (p) => (<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="currentColor" strokeWidth="1.8" {...p}><path d="M15 5 8 12l7 7" strokeLinecap="round" strokeLinejoin="round" /></svg>),
  menu: (p) => (<svg viewBox="0 0 24 24" width="22" height="22" fill="none" stroke="currentColor" strokeWidth="1.8" {...p}><path d="M4 7h16M4 12h16M4 17h16" strokeLinecap="round" /></svg>),
  close: (p) => (<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="currentColor" strokeWidth="2" {...p}><path d="M6 6l12 12M18 6 6 18" strokeLinecap="round" /></svg>),
  truck: (p) => (<svg viewBox="0 0 24 24" width="22" height="22" fill="none" stroke="currentColor" strokeWidth="1.7" {...p}><rect x="2.5" y="7" width="12" height="9" rx="1.2" /><path d="M14.5 10h3.6L21 13.2V16h-6.5" strokeLinejoin="round" /><circle cx="7" cy="18" r="1.8" /><circle cx="17" cy="18" r="1.8" /></svg>),
  star: (p) => (<svg viewBox="0 0 24 24" width="16" height="16" fill="currentColor" {...p}><path d="M12 2.5l2.9 6.1 6.6.7-4.9 4.6 1.3 6.6L12 17.5l-5.9 3-1.3-6.6-4.9-4.6 6.6-.7L12 2.5Z" /></svg>),
  check: (p) => (<svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" strokeWidth="2.2" {...p}><path d="m5 13 4.5 4.5L19 7" strokeLinecap="round" strokeLinejoin="round" /></svg>),
  phone: (p) => (<svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" strokeWidth="1.8" {...p}><path d="M6.5 3.5h3l1.5 4.5-2.2 1.8a13.5 13.5 0 0 0 6.4 6.4l1.8-2.2 4.5 1.5v3a2 2 0 0 1-2.1 2C11.6 20 4 12.4 4.5 5.6a2 2 0 0 1 2-2.1Z" strokeLinejoin="round" /></svg>),
  location: (p) => (<svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" strokeWidth="1.8" {...p}><path d="M12 21s7-6.5 7-11.5a7 7 0 1 0-14 0C5 14.5 12 21 12 21Z" strokeLinejoin="round" /><circle cx="12" cy="9.5" r="2.4" /></svg>),
  tiktok: (p) => (<svg viewBox="0 0 24 24" width="18" height="18" fill="currentColor" {...p}><path d="M16.6 3c.4 2 1.7 3.4 3.9 3.6v2.7c-1.4.1-2.7-.3-3.9-1.1v6.6c0 3.3-2.4 5.4-5.3 5.4-2.9 0-5.2-2.2-5.2-5.2 0-3 2.6-5.3 5.7-5 .2 0 .4 0 .6.1v2.8a2.6 2.6 0 0 0-.9-.2c-1.5 0-2.7 1.2-2.7 2.6 0 1.5 1.2 2.6 2.7 2.6 1.5 0 2.8-1.1 2.8-3V3h2.3Z" /></svg>),
  lock: (p) => (<svg viewBox="0 0 24 24" width="15" height="15" fill="none" stroke="currentColor" strokeWidth="1.8" {...p}><rect x="5" y="10.5" width="14" height="9" rx="1.5" /><path d="M8 10.5V8a4 4 0 0 1 8 0v2.5" strokeLinecap="round" /></svg>),
  plus: (p) => (<svg viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="currentColor" strokeWidth="2.2" {...p}><path d="M12 5v14M5 12h14" strokeLinecap="round" /></svg>),
  trash: (p) => (<svg viewBox="0 0 24 24" width="16" height="16" fill="none" stroke="currentColor" strokeWidth="1.8" {...p}><path d="M4 7h16M9 7V5a1 1 0 0 1 1-1h4a1 1 0 0 1 1 1v2m-8 0 1 13a1 1 0 0 0 1 1h6a1 1 0 0 0 1-1l1-13" strokeLinecap="round" strokeLinejoin="round" /></svg>),
};

/* ---------- APP ---------- */
export default function App() {
  const [view, setView] = useState("shop"); // shop | product | admin-login | admin
  const [menuOpen, setMenuOpen] = useState(false);
  const [search, setSearch] = useState("");
  const [selectedId, setSelectedId] = useState(null);
  const [products, setProducts] = useState([]);
  const [loading, setLoading] = useState(true);
  const [logoUrl, setLogoUrl] = useState(null);
  const [pwd, setPwd] = useState("");
  const [pwdError, setPwdError] = useState(false);
  const [editingProduct, setEditingProduct] = useState(null);

  useEffect(() => {
    (async () => {
      const stored = await fetchProducts();
      if (stored && stored.length) {
        setProducts(stored);
      } else if (stored) {
        await seedProducts(DEFAULT_PRODUCTS);
        const reloaded = await fetchProducts();
        setProducts(reloaded || []);
      }
      setLoading(false);
    })();
  }, []);

  useEffect(() => {
    (async () => {
      const url = await fetchSiteSetting("logo_url");
      if (url) setLogoUrl(url);
    })();
  }, []);

  const scrollTo = (id) => {
    setMenuOpen(false);
    setView("shop");
    setTimeout(() => document.getElementById(id)?.scrollIntoView({ behavior: "smooth", block: "start" }), 30);
  };

  function openProduct(id) {
    setSelectedId(id);
    setView("product");
    window.scrollTo({ top: 0 });
  }

  function tryAdminLogin() {
    if (pwd === ADMIN_PASSWORD) {
      setPwdError(false);
      setPwd("");
      setView("admin");
    } else {
      setPwdError(true);
    }
  }

  async function handleUpdateLogo(url) {
    setLogoUrl(url);
    await setSiteSetting("logo_url", url);
  }

  async function handleSaveProduct(p) {
    if (p.id) {
      const { id, ...rest } = p;
      await updateProductRow(id, rest);
      setProducts((prev) => prev.map((x) => (x.id === p.id ? p : x)));
    } else {
      const { id, ...rest } = p;
      const saved = await insertProductRow(rest);
      if (saved) setProducts((prev) => [...prev, saved]);
    }
    setEditingProduct(null);
  }

  async function handleDeleteProduct(id) {
    setProducts((prev) => prev.filter((p) => p.id !== id));
    await deleteProductRow(id);
  }

  const searching = search.trim().length > 0;
  const searchResults = searching
    ? products.filter((p) => p.name.toLowerCase().includes(search.trim().toLowerCase()))
    : [];

  const selected = products.find((p) => p.id === selectedId);

  if (loading) {
    return (
      <div style={{ minHeight: "100vh", display: "flex", alignItems: "center", justifyContent: "center", fontFamily: "'Inter', sans-serif" }}>
        Chargement…
      </div>
    );
  }

  return (
    <div className="page">
      <GlobalStyle />

      <header className="header">
        <div className="container header-inner">
          <div className="logo" onClick={() => scrollTo("accueil")}>
            {logoUrl ? <img src={logoUrl} alt="STAR DAV" className="logo-img" /> : <>STAR<span>DAV</span></>}
          </div>
          <nav className="nav-desktop">
            <button onClick={() => scrollTo("accueil")}>Accueil</button>
            <button onClick={() => scrollTo("boutique")}>Boutique</button>
            <button onClick={() => scrollTo("categories")}>Catégories</button>
            <button onClick={() => scrollTo("apropos")}>À propos</button>
            <button onClick={() => scrollTo("contact")}>Contact</button>
          </nav>
          <div className="header-actions">
            <a href={waLink("Bonjour STAR DAV, je souhaite passer une commande.")} target="_blank" rel="noreferrer" className="btn btn-primary btn-sm">
              <Icon.chat style={{ width: 15, height: 15 }} /> Commander
            </a>
            <button className="menu-btn" onClick={() => setMenuOpen(true)} aria-label="Menu"><Icon.menu /></button>
          </div>
        </div>
      </header>

      {menuOpen && (
        <div className="mobile-menu">
          <button className="close-btn" onClick={() => setMenuOpen(false)}><Icon.close /></button>
          <button onClick={() => scrollTo("accueil")}>Accueil</button>
          <button onClick={() => scrollTo("boutique")}>Boutique</button>
          <button onClick={() => scrollTo("categories")}>Catégories</button>
          <button onClick={() => scrollTo("apropos")}>À propos</button>
          <button onClick={() => scrollTo("contact")}>Contact</button>
        </div>
      )}

      {view === "shop" && (
        <>
          <section id="accueil" className="hero">
            <div className="container hero-inner">
              <div className="eyebrow"><Icon.star /> La référence des stars !</div>
              {logoUrl ? (
                <img src={logoUrl} alt="STAR DAV" className="hero-logo" />
              ) : (
                <h1>STAR<span>DAV</span></h1>
              )}
              <p className="hero-desc">
                Découvrez notre sélection de vêtements, chaussures, maillots et accessoires tendance.
              </p>
              <div className="hero-actions">
                <button className="btn btn-outline" onClick={() => scrollTo("boutique")}>Découvrir les produits</button>
                <a href={waLink("Bonjour STAR DAV, je souhaite passer une commande.")} target="_blank" rel="noreferrer" className="btn btn-primary">
                  <Icon.chat style={{ width: 16, height: 16 }} /> Commander sur WhatsApp
                </a>
              </div>
            </div>
          </section>

          <section id="categories" className="section">
            <div className="container">
              <div className="grid-categories">
                {CATEGORIES.map((c) => (
                  <button key={c.key} className="category-card" onClick={() => scrollTo("cat-" + c.key)}>
                    <div className="category-emoji">{c.emoji}</div>
                    <div>{c.label}</div>
                  </button>
                ))}
              </div>
            </div>
          </section>

          {/* ---------- BOUTIQUE : tout est visible directement, regroupé par catégorie ---------- */}
          <section id="boutique" className="section alt">
            <div className="container">
              <div className="eyebrow center"><Icon.star /> Boutique</div>
              <h2 className="section-title center">Notre sélection</h2>

              <div className="search-bar">
                <Icon.search style={{ color: "#9a9a9a" }} />
                <input value={search} onChange={(e) => setSearch(e.target.value)} placeholder="Rechercher un produit…" />
              </div>

              {searching ? (
                <ProductGrid products={searchResults} onOpen={openProduct} emptyText="Aucun produit ne correspond à votre recherche." />
              ) : (
                CATEGORIES.map((c) => {
                  const items = products.filter((p) => p.category === c.key && p.available !== false);
                  if (items.length === 0) return null;
                  return (
                    <div key={c.key} id={"cat-" + c.key} className="category-section">
                      <h3 className="category-heading">{c.emoji} {c.label}</h3>
                      <ProductGrid products={items} onOpen={openProduct} />
                    </div>
                  );
                })
              )}
            </div>
          </section>

          <section className="section">
            <div className="container">
              <div className="delivery-box">
                <Icon.truck className="delivery-ic" />
                <div>
                  <h3>Livraison</h3>
                  <p>
                    Livraison disponible à Ouagadougou et selon les zones desservies.
                    Les frais de livraison sont communiqués selon la zone — contactez-nous
                    sur WhatsApp pour connaître le tarif exact vers chez vous.
                  </p>
                </div>
              </div>
            </div>
          </section>

          <section id="apropos" className="section alt">
            <div className="container">
              <div className="eyebrow center"><Icon.star /> Pourquoi nous choisir</div>
              <h2 className="section-title center">Pourquoi choisir STAR DAV ?</h2>
              <div className="grid-avantages">
                {["Produits de qualité", "Prix accessibles", "Livraison disponible", "Service client à l'écoute"].map((a) => (
                  <div key={a} className="avantage-item">
                    <span className="check-circle"><Icon.check style={{ width: 15, height: 15 }} /></span>
                    <span>{a}</span>
                  </div>
                ))}
              </div>
            </div>
          </section>

          <section id="contact" className="section">
            <div className="container">
              <div className="eyebrow center"><Icon.star /> Contact</div>
              <h2 className="section-title center">Où et comment nous joindre</h2>
              <div className="contact-grid">
                <div className="contact-item">
                  <Icon.phone className="contact-ic" />
                  <div><div className="contact-label">Téléphone</div><div className="contact-value">{PHONE_DISPLAY}</div></div>
                </div>
                <div className="contact-item">
                  <Icon.chat className="contact-ic" />
                  <div><div className="contact-label">WhatsApp</div><div className="contact-value">{PHONE_DISPLAY}</div></div>
                </div>
                <a className="contact-item" href={TIKTOK_URL} target="_blank" rel="noreferrer">
                  <Icon.tiktok className="contact-ic" />
                  <div><div className="contact-label">TikTok</div><div className="contact-value">{TIKTOK}</div></div>
                </a>
                <div className="contact-item">
                  <Icon.location className="contact-ic" />
                  <div><div className="contact-label">Localisation</div><div className="contact-value">{LOCATION}</div></div>
                </div>
              </div>
              <a href={waLink("Bonjour STAR DAV, je souhaite avoir plus d'informations.")} target="_blank" rel="noreferrer" className="btn btn-primary" style={{ marginTop: 24 }}>
                <Icon.chat style={{ width: 16, height: 16 }} /> Écrire sur WhatsApp
              </a>

              <div style={{ textAlign: "center", marginTop: 30 }}>
                <button onClick={() => setView("admin-login")} style={{ background: "none", border: "none", color: "#9a9a9a", fontSize: 11.5, display: "inline-flex", alignItems: "center", gap: 5 }}>
                  <Icon.lock /> Espace admin
                </button>
              </div>
            </div>
          </section>
        </>
      )}

      {view === "product" && selected && <ProductDetail product={selected} onBack={() => setView("shop")} />}

      {view === "admin-login" && (
        <AdminLogin pwd={pwd} setPwd={setPwd} error={pwdError} onSubmit={tryAdminLogin} onBack={() => setView("shop")} />
      )}

      {view === "admin" && (
        <AdminView
          products={products}
          onEdit={setEditingProduct}
          onDelete={handleDeleteProduct}
          onNew={() => setEditingProduct({ name: "", category: CATEGORIES[0].key, price: "", image: "", description: "", sizes: "", available: true })}
          onLogout={() => setView("shop")}
          logoUrl={logoUrl}
          onUpdateLogo={handleUpdateLogo}
        />
      )}

      {editingProduct && (
        <ProductEditor product={editingProduct} onSave={handleSaveProduct} onCancel={() => setEditingProduct(null)} />
      )}

      {view === "shop" && (
        <footer className="footer">
          <div className="container footer-inner">
            <div>
              <div className="logo footer-logo">{logoUrl ? <img src={logoUrl} alt="STAR DAV" className="logo-img" /> : <>STAR<span>DAV</span></>}</div>
              <div className="footer-slogan">« La référence des stars ! »</div>
            </div>
            <div className="footer-info">
              <div>{LOCATION}</div>
              <div>{PHONE_DISPLAY}</div>
              <div>TikTok : {TIKTOK}</div>
            </div>
          </div>
          <div className="footer-bottom">© {new Date().getFullYear()} STAR DAV</div>
        </footer>
      )}

      <a href={waLink("Bonjour STAR DAV, je souhaite avoir plus d'informations.")} target="_blank" rel="noreferrer" className="floating-wa" aria-label="Écrire sur WhatsApp">
        <Icon.chat style={{ width: 26, height: 26 }} />
      </a>
    </div>
  );
}

/* ---------- Grille de produits réutilisable ---------- */
function ProductGrid({ products, onOpen, emptyText }) {
  if (products.length === 0) {
    return <div style={{ textAlign: "center", color: "#9a9a9a", padding: "30px 0" }}>{emptyText}</div>;
  }
  return (
    <div className="grid-products">
      {products.map((p) => (
        <button key={p.id} className="product-card" onClick={() => onOpen(p.id)}>
          <div className="product-img">
            <img src={p.image} alt={p.name} />
          </div>
          <div className="product-info">
            <div className="product-name">{p.name}</div>
            <div className="product-price">{fmt(p.price)}</div>
          </div>
        </button>
      ))}
    </div>
  );
}

/* ---------- FICHE PRODUIT ---------- */
function ProductDetail({ product, onBack }) {
  const msg = `Bonjour STAR DAV, je suis intéressé(e) par le produit ${product.name}. Pouvez-vous me confirmer sa disponibilité et les conditions de livraison ?`;
  const sizes = sizesToArray(product.sizes);
  return (
    <div className="container product-detail">
      <button className="back-btn" onClick={onBack}><Icon.back /> Retour</button>
      <div className="product-detail-grid">
        <div className="product-detail-img">
          <img src={product.image} alt={product.name} />
        </div>
        <div className="product-detail-info">
          <div className="eyebrow">{CATEGORIES.find((c) => c.key === product.category)?.label}</div>
          <h1>{product.name}</h1>
          <div className="product-detail-price">{fmt(product.price)}</div>
          <div className="availability"><span className="dot" /> Disponible</div>
          <p className="product-detail-desc">{product.description}</p>
          {sizes.length > 0 && (
            <div className="sizes">
              <div className="sizes-label">Tailles disponibles</div>
              <div className="sizes-list">{sizes.map((s) => <span key={s} className="size-chip">{s}</span>)}</div>
            </div>
          )}
          <a href={waLink(msg)} target="_blank" rel="noreferrer" className="btn btn-primary btn-full">
            <Icon.chat style={{ width: 17, height: 17 }} /> Commander sur WhatsApp
          </a>
        </div>
      </div>
    </div>
  );
}

/* ---------- ADMIN LOGIN ---------- */
function AdminLogin({ pwd, setPwd, error, onSubmit, onBack }) {
  return (
    <div style={{ minHeight: "100vh", display: "flex", flexDirection: "column", justifyContent: "center", padding: "20px 28px", fontFamily: "'Inter', sans-serif" }}>
      <button onClick={onBack} style={{ background: "none", border: "none", display: "flex", marginBottom: 30 }}><Icon.back /></button>
      <div style={{ display: "flex", justifyContent: "center", marginBottom: 18 }}>
        <div style={{ width: 52, height: 52, borderRadius: "50%", background: "#0A0A0A", color: "#fff", display: "flex", alignItems: "center", justifyContent: "center" }}>
          <Icon.lock />
        </div>
      </div>
      <h2 style={{ fontFamily: "'Anton', sans-serif", fontSize: 19, textAlign: "center", margin: "0 0 6px" }}>Espace admin</h2>
      <p style={{ fontSize: 12.5, color: "#9a9a9a", textAlign: "center", margin: "0 0 24px" }}>Réservé à STAR DAV</p>
      <input
        type="password"
        value={pwd}
        onChange={(e) => setPwd(e.target.value)}
        onKeyDown={(e) => e.key === "Enter" && onSubmit()}
        placeholder="Mot de passe"
        style={{ border: `1.5px solid ${error ? "#c0392b" : "#0A0A0A33"}`, borderRadius: 12, padding: "14px 16px", fontSize: 14, marginBottom: error ? 8 : 18, textAlign: "center" }}
      />
      {error && <div style={{ fontSize: 12, color: "#c0392b", textAlign: "center", marginBottom: 14 }}>Mot de passe incorrect.</div>}
      <button onClick={onSubmit} style={{ padding: "14px 0", borderRadius: 12, border: "none", background: "#A6114F", color: "#fff", fontSize: 13.5, fontWeight: 700 }}>
        Entrer
      </button>
    </div>
  );
}

/* ---------- ADMIN VIEW ---------- */
function AdminView({ products, onEdit, onDelete, onNew, onLogout, logoUrl, onUpdateLogo }) {
  const [confirmDelete, setConfirmDelete] = useState(null);
  const [uploadingLogo, setUploadingLogo] = useState(false);

  async function handleLogoFile(e) {
    const file = e.target.files?.[0];
    if (!file) return;
    setUploadingLogo(true);
    const url = await uploadImage(file, "branding");
    setUploadingLogo(false);
    if (url) onUpdateLogo(url);
    else alert("L'envoi du logo a échoué, réessayez.");
  }

  return (
    <div style={{ padding: "24px 20px 30px", fontFamily: "'Inter', sans-serif", maxWidth: 600, margin: "0 auto" }}>
      <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between", marginBottom: 20 }}>
        <h2 style={{ fontFamily: "'Anton', sans-serif", fontSize: 19, margin: 0 }}>Mes produits</h2>
        <button onClick={() => setConfirmDelete({ logout: true })} style={{ background: "none", border: "1.5px solid #0A0A0A33", borderRadius: 20, padding: "6px 14px", fontSize: 11.5 }}>
          Quitter
        </button>
      </div>

      <div style={{ background: "#fff", border: "1px solid #eee", borderRadius: 14, padding: 14, marginBottom: 16, display: "flex", alignItems: "center", gap: 12 }}>
        <div style={{ width: 52, height: 52, borderRadius: 10, overflow: "hidden", background: "#F4F4F4", flex: "0 0 auto", display: "flex", alignItems: "center", justifyContent: "center" }}>
          {logoUrl ? <img src={logoUrl} alt="" style={{ width: "100%", height: "100%", objectFit: "contain" }} /> : <span style={{ fontSize: 10, color: "#9a9a9a" }}>Logo</span>}
        </div>
        <div style={{ flex: 1 }}>
          <div style={{ fontSize: 12.5, fontWeight: 600, marginBottom: 6 }}>Logo du site</div>
          <label style={{ display: "inline-block", padding: "8px 14px", borderRadius: 10, border: "1.5px solid #0A0A0A", background: "#fff", fontSize: 11.5, fontWeight: 700, cursor: "pointer" }}>
            {uploadingLogo ? "Envoi…" : "Changer le logo"}
            <input type="file" accept="image/*" onChange={handleLogoFile} style={{ display: "none" }} />
          </label>
        </div>
      </div>

      <button
        onClick={onNew}
        style={{ width: "100%", padding: "12px 0", borderRadius: 12, border: "1.5px dashed #0A0A0A44", background: "#fff", fontSize: 12.5, fontWeight: 600, marginBottom: 16, display: "flex", alignItems: "center", justifyContent: "center", gap: 6 }}
      >
        <Icon.plus /> Ajouter un produit
      </button>

      {products.length === 0 && <div style={{ textAlign: "center", color: "#9a9a9a", fontSize: 13, padding: "30px 20px" }}>Aucun produit pour l'instant.</div>}

      <div style={{ display: "flex", flexDirection: "column", gap: 10 }}>
        {products.map((p) => (
          <div key={p.id} style={{ display: "flex", gap: 10, background: "#fff", border: "1px solid #eee", borderRadius: 14, padding: 10, alignItems: "center" }}>
            <div style={{ width: 46, height: 46, borderRadius: 10, overflow: "hidden", background: "#F4F4F4", flex: "0 0 auto" }}>
              {p.image && <img src={p.image} alt="" style={{ width: "100%", height: "100%", objectFit: "cover" }} />}
            </div>
            <div style={{ flex: 1, minWidth: 0 }}>
              <div style={{ fontSize: 13, fontWeight: 600, whiteSpace: "nowrap", overflow: "hidden", textOverflow: "ellipsis" }}>{p.name}</div>
              <div style={{ fontSize: 12, color: "#6b6b6b" }}>{fmt(p.price)} · {CATEGORIES.find((c) => c.key === p.category)?.label || p.category} {p.available === false ? "· Masqué" : ""}</div>
            </div>
            <button onClick={() => onEdit(p)} style={{ background: "none", border: "1.5px solid #0A0A0A33", borderRadius: 8, padding: "6px 10px", fontSize: 11 }}>Modifier</button>
            <button onClick={() => setConfirmDelete({ id: p.id, name: p.name })} style={{ background: "none", border: "none", color: "#c0392b" }}><Icon.trash /></button>
          </div>
        ))}
      </div>

      {confirmDelete && (
        <div style={{ position: "fixed", inset: 0, background: "rgba(10,10,10,0.5)", display: "flex", alignItems: "center", justifyContent: "center", zIndex: 60, padding: 24 }}>
          <div style={{ background: "#fff", borderRadius: 16, padding: "22px 20px", maxWidth: 300, width: "100%" }}>
            <p style={{ fontSize: 14, lineHeight: 1.5, margin: "0 0 18px" }}>
              {confirmDelete.logout ? "Voulez-vous vraiment quitter l'espace admin ?" : `Supprimer "${confirmDelete.name}" ? Cette action est définitive.`}
            </p>
            <div style={{ display: "flex", gap: 10 }}>
              <button onClick={() => setConfirmDelete(null)} style={{ flex: 1, padding: "12px 0", borderRadius: 10, border: "1.5px solid #0A0A0A33", background: "#fff", fontSize: 13, fontWeight: 600 }}>Annuler</button>
              <button
                onClick={() => {
                  if (confirmDelete.logout) onLogout();
                  else onDelete(confirmDelete.id);
                  setConfirmDelete(null);
                }}
                style={{ flex: 1, padding: "12px 0", borderRadius: 10, border: "none", background: confirmDelete.logout ? "#0A0A0A" : "#c0392b", color: "#fff", fontSize: 13, fontWeight: 700 }}
              >
                {confirmDelete.logout ? "Quitter" : "Supprimer"}
              </button>
            </div>
          </div>
        </div>
      )}
    </div>
  );
}

/* ---------- ÉDITEUR DE PRODUIT (modal) ---------- */
function ProductEditor({ product, onSave, onCancel }) {
  const [form, setForm] = useState(product);
  const [uploading, setUploading] = useState(false);
  const set = (k) => (e) => setForm((f) => ({ ...f, [k]: e.target.value }));
  const inputStyle = { border: "1.5px solid #0A0A0A22", borderRadius: 10, padding: "11px 12px", fontSize: 13.5, width: "100%", fontFamily: "'Inter', sans-serif" };

  async function handleFile(e) {
    const file = e.target.files?.[0];
    if (!file) return;
    setUploading(true);
    const url = await uploadImage(file, "products");
    setUploading(false);
    if (url) setForm((f) => ({ ...f, image: url }));
    else alert("L'envoi de la photo a échoué, réessayez.");
  }

  return (
    <div style={{ position: "fixed", inset: 0, background: "rgba(10,10,10,0.5)", display: "flex", alignItems: "flex-end", zIndex: 50, fontFamily: "'Inter', sans-serif" }}>
      <div style={{ background: "#fff", borderRadius: "20px 20px 0 0", padding: "22px 20px 30px", width: "100%", maxHeight: "88vh", overflowY: "auto" }}>
        <h3 style={{ fontFamily: "'Anton', sans-serif", fontSize: 17, margin: "0 0 18px" }}>{product.id ? "Modifier le produit" : "Nouveau produit"}</h3>
        <div style={{ display: "flex", flexDirection: "column", gap: 12 }}>
          <div>
            <label style={{ fontSize: 11, color: "#9a9a9a", letterSpacing: 0.4, textTransform: "uppercase", display: "block", marginBottom: 5 }}>Nom</label>
            <input value={form.name} onChange={set("name")} style={inputStyle} />
          </div>
          <div>
            <label style={{ fontSize: 11, color: "#9a9a9a", letterSpacing: 0.4, textTransform: "uppercase", display: "block", marginBottom: 5 }}>Catégorie</label>
            <select value={form.category} onChange={set("category")} style={inputStyle}>
              {CATEGORIES.map((c) => <option key={c.key} value={c.key}>{c.emoji} {c.label}</option>)}
            </select>
          </div>
          <div>
            <label style={{ fontSize: 11, color: "#9a9a9a", letterSpacing: 0.4, textTransform: "uppercase", display: "block", marginBottom: 5 }}>Prix (FCFA)</label>
            <input type="number" value={form.price} onChange={set("price")} style={inputStyle} />
          </div>
          <div>
            <label style={{ fontSize: 11, color: "#9a9a9a", letterSpacing: 0.4, textTransform: "uppercase", display: "block", marginBottom: 5 }}>Photo</label>
            <div style={{ display: "flex", alignItems: "center", gap: 12 }}>
              <div style={{ width: 56, height: 56, borderRadius: 10, overflow: "hidden", background: "#F4F4F4", flex: "0 0 auto" }}>
                {form.image && <img src={form.image} alt="" style={{ width: "100%", height: "100%", objectFit: "cover" }} />}
              </div>
              <label style={{ padding: "10px 14px", borderRadius: 10, border: "1.5px solid #0A0A0A", background: "#fff", fontSize: 12, fontWeight: 700, cursor: "pointer" }}>
                {uploading ? "Envoi…" : "Choisir une photo"}
                <input type="file" accept="image/*" onChange={handleFile} style={{ display: "none" }} />
              </label>
            </div>
          </div>
          <div>
            <label style={{ fontSize: 11, color: "#9a9a9a", letterSpacing: 0.4, textTransform: "uppercase", display: "block", marginBottom: 5 }}>Description</label>
            <textarea rows={3} value={form.description} onChange={set("description")} style={{ ...inputStyle, resize: "vertical" }} />
          </div>
          <div>
            <label style={{ fontSize: 11, color: "#9a9a9a", letterSpacing: 0.4, textTransform: "uppercase", display: "block", marginBottom: 5 }}>Tailles (séparées par des virgules, laisser vide si non applicable)</label>
            <input value={form.sizes || ""} onChange={set("sizes")} placeholder="Ex : S,M,L,XL ou 38,39,40" style={inputStyle} />
          </div>
          <div>
            <label style={{ fontSize: 11, color: "#9a9a9a", letterSpacing: 0.4, textTransform: "uppercase", display: "block", marginBottom: 5 }}>Disponibilité</label>
            <div style={{ display: "flex", gap: 8 }}>
              <button type="button" onClick={() => setForm((f) => ({ ...f, available: true }))} style={{ flex: 1, padding: "10px 0", borderRadius: 10, border: "1.5px solid #0A0A0A", background: form.available !== false ? "#0A0A0A" : "#fff", color: form.available !== false ? "#fff" : "#0A0A0A", fontSize: 12.5, fontWeight: 700 }}>Visible</button>
              <button type="button" onClick={() => setForm((f) => ({ ...f, available: false }))} style={{ flex: 1, padding: "10px 0", borderRadius: 10, border: "1.5px solid #0A0A0A", background: form.available === false ? "#0A0A0A" : "#fff", color: form.available === false ? "#fff" : "#0A0A0A", fontSize: 12.5, fontWeight: 700 }}>Masqué</button>
            </div>
          </div>
        </div>
        <div style={{ display: "flex", gap: 10, marginTop: 20 }}>
          <button onClick={onCancel} style={{ flex: 1, padding: "13px 0", borderRadius: 12, border: "1.5px solid #0A0A0A33", background: "#fff", fontSize: 13, fontWeight: 600 }}>Annuler</button>
          <button onClick={() => onSave({ ...form, price: Number(form.price) || 0 })} style={{ flex: 1, padding: "13px 0", borderRadius: 12, border: "none", background: "#A6114F", color: "#fff", fontSize: 13, fontWeight: 700 }}>Enregistrer</button>
        </div>
      </div>
    </div>
  );
}

function GlobalStyle() {
  return (
    <style>{`
      @import url('https://fonts.googleapis.com/css2?family=Anton&family=Inter:wght@400;500;600;700;800&display=swap');

      * { box-sizing: border-box; margin: 0; padding: 0; }
      html { scroll-behavior: smooth; }
      body { -webkit-font-smoothing: antialiased; }
      button { font-family: inherit; cursor: pointer; border: none; background: none; }
      a { text-decoration: none; color: inherit; }

      .page { font-family: 'Inter', -apple-system, sans-serif; color: #0A0A0A; background: #fff; overflow-x: hidden; }
      .container { max-width: 1080px; margin: 0 auto; padding: 0 20px; }

      .eyebrow { display: inline-flex; align-items: center; gap: 6px; font-size: 11.5px; font-weight: 700; letter-spacing: 1.5px; color: #A6114F; text-transform: uppercase; margin-bottom: 10px; }
      .eyebrow.center { display: flex; justify-content: center; width: 100%; }
      .section-title { font-family: 'Anton', sans-serif; font-weight: 400; font-size: 26px; letter-spacing: 0.3px; margin-bottom: 30px; }
      .section-title.center { text-align: center; }
      .section { padding: 56px 0; }
      .section.alt { background: #FAFAFA; }

      .btn { display: inline-flex; align-items: center; justify-content: center; gap: 8px; padding: 13px 22px; border-radius: 10px; font-weight: 700; font-size: 13.5px; }
      .btn-primary { background: #A6114F; color: #fff; }
      .btn-outline { background: #fff; color: #0A0A0A; border: 1.5px solid #0A0A0A; }
      .btn-sm { padding: 9px 14px; font-size: 12px; }
      .btn-full { width: 100%; padding: 15px 0; font-size: 14px; }

      .header { position: sticky; top: 0; background: rgba(255,255,255,0.96); backdrop-filter: blur(6px); z-index: 40; border-bottom: 1px solid #eee; }
      .header-inner { display: flex; align-items: center; justify-content: space-between; padding-top: 14px; padding-bottom: 14px; }
      .logo { font-family: 'Anton', sans-serif; font-size: 19px; letter-spacing: 0.5px; cursor: pointer; display: flex; align-items: center; }
      .logo span { color: #A6114F; }
      .logo-img { height: 34px; width: auto; }
      .nav-desktop { display: none; gap: 24px; }
      .nav-desktop button { font-size: 13px; font-weight: 600; }
      .header-actions { display: flex; align-items: center; gap: 10px; }
      .menu-btn { display: flex; }

      .mobile-menu { position: fixed; inset: 0; background: #fff; z-index: 60; display: flex; flex-direction: column; padding: 20px; gap: 4px; }
      .mobile-menu .close-btn { align-self: flex-end; margin-bottom: 20px; }
      .mobile-menu button { text-align: left; font-family: 'Anton', sans-serif; font-size: 22px; padding: 14px 4px; border-bottom: 1px solid #f0f0f0; }

      .hero { background: #0A0A0A; color: #fff; padding: 60px 0 54px; text-align: center; }
      .hero .eyebrow { color: #E8407B; justify-content: center; width: 100%; }
      .hero h1 { font-family: 'Anton', sans-serif; font-size: 52px; letter-spacing: 1px; line-height: 1; margin-bottom: 16px; }
      .hero h1 span { color: #A6114F; }
      .hero-logo { max-width: 220px; width: 100%; height: auto; margin: 0 auto 16px; }
      .hero-desc { font-size: 14.5px; color: #ccc; max-width: 420px; margin: 0 auto 26px; line-height: 1.6; }
      .hero-actions { display: flex; flex-wrap: wrap; gap: 12px; justify-content: center; }
      .hero .btn-outline { background: transparent; color: #fff; border-color: #fff; }

      .grid-categories { display: grid; grid-template-columns: 1fr 1fr; gap: 12px; margin-top: -30px; }
      .category-card { background: #fff; border: 1.5px solid #eee; border-radius: 16px; padding: 20px 10px; display: flex; flex-direction: column; align-items: center; gap: 8px; font-size: 12.5px; font-weight: 700; box-shadow: 0 4px 16px rgba(0,0,0,0.06); }
      .category-emoji { font-size: 26px; }

      .search-bar { display: flex; align-items: center; gap: 10px; background: #fff; border: 1.5px solid #eee; border-radius: 12px; padding: 12px 16px; margin-bottom: 26px; }
      .search-bar input { flex: 1; border: none; outline: none; font-size: 14px; font-family: inherit; }

      .category-section { margin-bottom: 36px; }
      .category-heading { font-family: 'Anton', sans-serif; font-size: 18px; letter-spacing: 0.3px; margin-bottom: 14px; }

      .grid-products { display: grid; grid-template-columns: 1fr 1fr; gap: 16px; }
      .product-card { background: #fff; border: 1px solid #eee; border-radius: 16px; overflow: hidden; text-align: left; }
      .product-img { width: 100%; aspect-ratio: 1/1; background: #F4F4F4; position: relative; overflow: hidden; }
      .product-img img { width: 100%; height: 100%; object-fit: cover; }
      .product-info { padding: 12px 14px 16px; }
      .product-name { font-size: 13.5px; font-weight: 600; margin-bottom: 5px; }
      .product-price { font-family: 'Anton', sans-serif; font-size: 15px; color: #A6114F; }

      .delivery-box { display: flex; gap: 16px; background: #0A0A0A; color: #fff; border-radius: 18px; padding: 24px; align-items: flex-start; }
      .delivery-ic { color: #E8407B; flex: 0 0 auto; margin-top: 3px; }
      .delivery-box h3 { font-family: 'Anton', sans-serif; font-size: 17px; margin-bottom: 8px; letter-spacing: 0.3px; }
      .delivery-box p { font-size: 13px; color: #ccc; line-height: 1.6; }

      .grid-avantages { display: flex; flex-direction: column; gap: 12px; max-width: 480px; margin: 0 auto; }
      .avantage-item { display: flex; align-items: center; gap: 14px; background: #fff; border: 1px solid #eee; border-radius: 14px; padding: 14px 16px; font-size: 14px; font-weight: 600; }
      .check-circle { width: 30px; height: 30px; border-radius: 50%; background: #A6114F; color: #fff; display: flex; align-items: center; justify-content: center; flex: 0 0 auto; }

      .contact-grid { display: grid; grid-template-columns: 1fr; gap: 12px; max-width: 480px; margin: 0 auto; }
      .contact-item { display: flex; align-items: center; gap: 14px; background: #fff; border: 1px solid #eee; border-radius: 14px; padding: 14px 16px; }
      .contact-ic { color: #A6114F; flex: 0 0 auto; }
      .contact-label { font-size: 10px; text-transform: uppercase; letter-spacing: 0.8px; color: #9a9a9a; font-weight: 600; margin-bottom: 2px; }
      .contact-value { font-size: 13.5px; font-weight: 700; }

      .product-detail { padding: 24px 20px 60px; }
      .back-btn { display: inline-flex; align-items: center; gap: 6px; font-weight: 700; font-size: 13px; margin-bottom: 20px; }
      .product-detail-grid { display: flex; flex-direction: column; gap: 26px; }
      .product-detail-img { position: relative; width: 100%; aspect-ratio: 1/1; border-radius: 18px; overflow: hidden; background: #F4F4F4; }
      .product-detail-img img { width: 100%; height: 100%; object-fit: cover; }
      .product-detail-info h1 { font-family: 'Anton', sans-serif; font-size: 28px; margin: 4px 0 10px; letter-spacing: 0.3px; }
      .product-detail-price { font-family: 'Anton', sans-serif; font-size: 24px; color: #A6114F; margin-bottom: 12px; }
      .availability { display: inline-flex; align-items: center; gap: 7px; font-size: 12.5px; font-weight: 700; color: #1a8a4c; margin-bottom: 18px; }
      .dot { width: 8px; height: 8px; border-radius: 50%; background: #1a8a4c; }
      .product-detail-desc { font-size: 14px; color: #4a4a4a; line-height: 1.7; margin-bottom: 20px; }
      .sizes-label { font-size: 11px; text-transform: uppercase; letter-spacing: 0.8px; color: #9a9a9a; font-weight: 700; margin-bottom: 8px; }
      .sizes-list { display: flex; flex-wrap: wrap; gap: 8px; margin-bottom: 22px; }
      .size-chip { border: 1.5px solid #ddd; border-radius: 8px; padding: 6px 12px; font-size: 12.5px; font-weight: 600; }

      .footer { background: #0A0A0A; color: #fff; padding: 40px 0 0; }
      .footer-inner { display: flex; flex-direction: column; gap: 20px; padding-bottom: 26px; }
      .footer-logo { color: #fff; }
      .footer-logo span { color: #E8407B; }
      .footer-slogan { font-size: 12px; color: #999; margin-top: 4px; }
      .footer-info { display: flex; flex-direction: column; gap: 4px; font-size: 12.5px; color: #ccc; }
      .footer-bottom { border-top: 1px solid rgba(255,255,255,0.1); text-align: center; padding: 14px 20px; font-size: 11px; color: #777; }

      .floating-wa { position: fixed; bottom: 20px; right: 20px; width: 56px; height: 56px; border-radius: 50%; background: #25D366; color: #fff; display: flex; align-items: center; justify-content: center; box-shadow: 0 4px 14px rgba(0,0,0,0.25); z-index: 45; }

      @media (min-width: 640px) {
        .grid-categories { grid-template-columns: repeat(4, 1fr); }
        .grid-products { grid-template-columns: repeat(3, 1fr); }
        .contact-grid { grid-template-columns: 1fr 1fr; }
        .product-detail-grid { flex-direction: row; align-items: flex-start; }
        .product-detail-img { flex: 1; }
        .product-detail-info { flex: 1; }
      }
      @media (min-width: 860px) {
        .nav-desktop { display: flex; }
        .menu-btn { display: none; }
        .hero h1 { font-size: 64px; }
        .grid-products { grid-template-columns: repeat(4, 1fr); }
        .footer-inner { flex-direction: row; justify-content: space-between; }
      }
    `}</style>
  );
}
