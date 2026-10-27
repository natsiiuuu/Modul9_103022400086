//vite
import react from "@vitejs/plugin-react";
import { defineConfig } from "vite";
import tailwindcss from "@tailwindcss/vite";

// Proxy ini digunakan saat development agar browser tidak memanggil GamerPower
// secara langsung. Untuk deployment, dipakai serverless proxy di folder api/.
export default defineConfig({
  plugins: [react(), tailwindcss()],
  server: {
    proxy: {
      "/api/gamerpower": {
        target: "https://www.gamerpower.com",
        changeOrigin: true,
        secure: true,
        rewrite: (path) => path.replace(/^\/api\/gamerpower/, "/api"),
      },
    },
  },
});


//gamerpower

const API_BASE = "/api/gamerpower";

export async function getGiveaways({ signal } = {}) {
  const response = await fetch(`${API_BASE}/giveaways?sort-by=date`, { signal });

  if (!response.ok) {
    throw new Error(`Gagal mengambil data GamerPower (HTTP ${response.status}).`);
  }

  const data = await response.json();

  // GamerPower biasanya mengembalikan array. Respons non-array menandakan
  // respons status/pesan API yang tidak bisa ditampilkan sebagai daftar.
  if (!Array.isArray(data)) {
    throw new Error(
      data?.status_message ||
        data?.message ||
        "Format data dari GamerPower tidak sesuai."
    );
  }

  return data.filter((item) => {
    const status = String(item?.status ?? "Active").toLowerCase();
    return status === "active";
  });
}


//freegames
import { useEffect, useMemo, useState } from "react";
import { getGiveaways } from "../services/gamerpower";

const PLATFORM_OPTIONS = [
  { value: "all", label: "Semua platform" },
  { value: "pc", label: "PC" },
  { value: "steam", label: "Steam" },
  { value: "epic games store", label: "Epic Games Store" },
  { value: "gog", label: "GOG" },
  { value: "playstation 4", label: "PlayStation 4" },
  { value: "playstation 5", label: "PlayStation 5" },
  { value: "xbox one", label: "Xbox One" },
  { value: "xbox series x/s", label: "Xbox Series X/S" },
  { value: "switch", label: "Nintendo Switch" },
  { value: "android", label: "Android" },
  { value: "ios", label: "iOS" },
  { value: "drm-free", label: "DRM-Free" },
  { value: "itch.io", label: "itch.io" },
];

const TYPE_OPTIONS = [
  { value: "all", label: "Semua tipe" },
  { value: "game", label: "Game" },
  { value: "loot", label: "Loot" },
  { value: "dlc", label: "DLC" },
  { value: "beta", label: "Beta / Playtest" },
];

function formatDate(value) {
  if (!value || value === "N/A") return "Tidak disebutkan";
  const date = new Date(String(value).replace(" ", "T") + (String(value).includes("Z") ? "" : "Z"));
  if (Number.isNaN(date.getTime())) return value;
  return new Intl.DateTimeFormat("id-ID", {
    day: "numeric",
    month: "short",
    year: "numeric",
    timeZone: "Asia/Jakarta",
  }).format(date);
}

function GiveawayCard({ giveaway }) {
  const platformText = giveaway.platforms || "Platform tidak disebutkan";
  const typeText = giveaway.type || "Giveaway";
  const title = giveaway.title || "Giveaway tanpa judul";
  const image = giveaway.thumbnail || giveaway.image;

  return (
    <article className="flex h-full flex-col overflow-hidden rounded-2xl border border-slate-800 bg-slate-900/80 transition hover:-translate-y-1 hover:border-emerald-400/60">
      <div className="relative aspect-[16/9] overflow-hidden bg-slate-800">
        {image ? (
          <img
            src={image}
            alt={`Gambar ${title}`}
            loading="lazy"
            className="h-full w-full object-cover"
          />
        ) : (
          <div className="flex h-full items-center justify-center text-sm text-slate-400">
            Gambar tidak tersedia
          </div>
        )}
        <span className="absolute left-3 top-3 rounded-full border border-emerald-300/30 bg-slate-950/90 px-3 py-1 text-xs font-semibold text-emerald-300">
          {typeText}
        </span>
      </div>

      <div className="flex flex-1 flex-col p-5">
        <h2 className="line-clamp-2 text-lg font-bold leading-snug text-white">{title}</h2>
        <p className="mt-3 line-clamp-3 flex-1 text-sm leading-6 text-slate-400">
          {giveaway.description || "Lihat halaman giveaway untuk informasi dan ketentuan klaim."}
        </p>

        <dl className="mt-4 space-y-2 text-sm">
          <div className="flex items-start justify-between gap-3">
            <dt className="shrink-0 text-slate-500">Platform</dt>
            <dd className="text-right text-slate-200">{platformText}</dd>
          </div>
          <div className="flex items-start justify-between gap-3">
            <dt className="shrink-0 text-slate-500">Nilai</dt>
            <dd className="text-right font-semibold text-emerald-300">
              {giveaway.worth && giveaway.worth !== "N/A" ? giveaway.worth : "Gratis"}
            </dd>
          </div>
          <div className="flex items-start justify-between gap-3">
            <dt className="shrink-0 text-slate-500">Berakhir</dt>
            <dd className="text-right text-slate-200">{formatDate(giveaway.end_date)}</dd>
          </div>
        </dl>

        <a
          href={giveaway.open_giveaway_url || giveaway.gamerpower_url || "https://www.gamerpower.com"}
          target="_blank"
          rel="noopener noreferrer"
          className="mt-5 inline-flex min-h-11 items-center justify-center rounded-xl bg-emerald-400 px-4 py-2.5 text-sm font-bold text-slate-950 transition hover:bg-emerald-300 focus:outline-none focus:ring-2 focus:ring-emerald-300 focus:ring-offset-2 focus:ring-offset-slate-900"
        >
          Lihat Giveaway <span className="sr-only">(buka di tab baru)</span><span aria-hidden="true" className="ml-2">↗</span>
        </a>
      </div>
    </article>
  );
}

function LoadingCards() {
  return (
    <div className="grid gap-5 sm:grid-cols-2 xl:grid-cols-3" aria-label="Memuat giveaway" aria-busy="true">
      {Array.from({ length: 6 }, (_, index) => (
        <div key={index} className="overflow-hidden rounded-2xl border border-slate-800 bg-slate-900">
          <div className="aspect-[16/9] animate-pulse bg-slate-800" />
          <div className="space-y-3 p-5">
            <div className="h-5 w-4/5 animate-pulse rounded bg-slate-800" />
            <div className="h-4 w-full animate-pulse rounded bg-slate-800" />
            <div className="h-4 w-2/3 animate-pulse rounded bg-slate-800" />
            <div className="h-10 w-full animate-pulse rounded-xl bg-slate-800" />
          </div>
        </div>
      ))}
    </div>
  );
}

export default function FreeGames() {
  const [giveaways, setGiveaways] = useState([]);
  const [platform, setPlatform] = useState("all");
  const [type, setType] = useState("all");
  const [status, setStatus] = useState("loading");
  const [errorMessage, setErrorMessage] = useState("");
  const [retryKey, setRetryKey] = useState(0);

  useEffect(() => {
    const controller = new AbortController();

    async function loadGiveaways() {
      setStatus("loading");
      setErrorMessage("");

      try {
        const data = await getGiveaways({ signal: controller.signal });
        setGiveaways(data);
        setStatus("success");
      } catch (error) {
        if (error.name === "AbortError") return;
        setErrorMessage(error.message || "Terjadi kesalahan saat memuat giveaway.");
        setStatus("error");
      }
    }

    loadGiveaways();
    return () => controller.abort();
  }, [retryKey]);

  const filteredGiveaways = useMemo(() => {
    return giveaways.filter((giveaway) => {
      const platforms = String(giveaway.platforms || "").toLowerCase();
      const giveawayType = String(giveaway.type || "").toLowerCase();

      const matchesPlatform =
        platform === "all" || platforms.includes(platform);
      const matchesType =
        type === "all" ||
        giveawayType === type ||
        (type === "beta" && giveawayType.includes("early access"));

      return matchesPlatform && matchesType;
    });
  }, [giveaways, platform, type]);

  return (
    <main className="min-h-screen bg-[#080a0e] px-4 py-10 text-slate-100 sm:px-6 lg:px-8">
      <div className="mx-auto max-w-7xl">
        <header className="mb-8 overflow-hidden rounded-3xl border border-slate-800 bg-gradient-to-br from-slate-900 via-slate-900 to-emerald-950/50 p-6 sm:p-9">
          <p className="mb-3 text-xs font-bold uppercase tracking-[0.22em] text-emerald-300">
            Gamer Wallet TZY · Free Games
          </p>
          <div className="flex flex-col justify-between gap-5 md:flex-row md:items-end">
            <div className="max-w-2xl">
              <h1 className="text-3xl font-black tracking-tight text-white sm:text-4xl">
                Temukan game gratis
              </h1>
              <p className="mt-3 max-w-xl text-sm leading-6 text-slate-400 sm:text-base">
                Pantau giveaway game, DLC, dan item gratis yang sedang aktif.
                Pilih platform dan tipe giveaway yang kamu cari.
              </p>
            </div>
            <div className="rounded-2xl border border-emerald-300/20 bg-emerald-300/10 px-5 py-4">
              <p className="text-sm text-slate-400">Giveaway aktif tersedia</p>
              <p className="mt-1 text-3xl font-extrabold text-emerald-300">
                {status === "success" ? filteredGiveaways.length : "—"}
              </p>
            </div>
          </div>
        </header>

        <section aria-label="Filter giveaway" className="mb-7 rounded-2xl border border-slate-800 bg-slate-900/70 p-4 sm:p-5">
          <div className="grid gap-4 sm:grid-cols-2">
            <div>
              <label htmlFor="free-games-platform" className="mb-2 block text-sm font-semibold text-slate-200">
                Platform
              </label>
              <select
                id="free-games-platform"
                value={platform}
                onChange={(event) => setPlatform(event.target.value)}
                className="min-h-11 w-full rounded-xl border border-slate-700 bg-slate-950 px-3 text-sm text-white outline-none focus:border-emerald-400 focus:ring-2 focus:ring-emerald-400/20"
              >
                {PLATFORM_OPTIONS.map((option) => (
                  <option key={option.value} value={option.value}>{option.label}</option>
                ))}
              </select>
            </div>
            <div>
              <label htmlFor="free-games-type" className="mb-2 block text-sm font-semibold text-slate-200">
                Tipe giveaway
              </label>
              <select
                id="free-games-type"
                value={type}
                onChange={(event) => setType(event.target.value)}
                className="min-h-11 w-full rounded-xl border border-slate-700 bg-slate-950 px-3 text-sm text-white outline-none focus:border-emerald-400 focus:ring-2 focus:ring-emerald-400/20"
              >
                {TYPE_OPTIONS.map((option) => (
                  <option key={option.value} value={option.value}>{option.label}</option>
                ))}
              </select>
            </div>
          </div>
          <div className="mt-4 flex flex-wrap items-center justify-between gap-3">
            <p className="text-sm text-slate-400" aria-live="polite">
              {status === "success"
                ? `${filteredGiveaways.length} giveaway ditemukan`
                : status === "loading"
                  ? "Sedang mengambil data giveaway…"
                  : "Data giveaway belum dapat ditampilkan"}
            </p>
            <button
              type="button"
              onClick={() => { setPlatform("all"); setType("all"); }}
              className="min-h-10 rounded-lg border border-slate-700 px-3 py-2 text-sm font-semibold text-slate-300 transition hover:border-emerald-400 hover:text-emerald-300 focus:outline-none focus:ring-2 focus:ring-emerald-400"
            >
              Reset filter
            </button>
          </div>
        </section>

        {status === "loading" && <LoadingCards />}

        {status === "error" && (
          <section role="alert" className="rounded-2xl border border-rose-400/30 bg-rose-950/20 p-6 text-center">
            <h2 className="text-lg font-bold text-white">Data giveaway gagal dimuat</h2>
            <p className="mx-auto mt-2 max-w-xl text-sm leading-6 text-rose-200/80">{errorMessage}</p>
            <p className="mx-auto mt-2 max-w-xl text-xs leading-5 text-slate-400">
              Periksa koneksi internet Anda, lalu coba lagi. Jika masih gagal, proxy GamerPower mungkin belum aktif.
            </p>
            <button
              type="button"
              onClick={() => setRetryKey((value) => value + 1)}
              className="mt-5 min-h-11 rounded-xl bg-emerald-400 px-5 py-2.5 text-sm font-bold text-slate-950 hover:bg-emerald-300 focus:outline-none focus:ring-2 focus:ring-emerald-300 focus:ring-offset-2 focus:ring-offset-slate-950"
            >
              Coba lagi
            </button>
          </section>
        )}

        {status === "success" && filteredGiveaways.length === 0 && (
          <section className="rounded-2xl border border-slate-800 bg-slate-900/60 px-5 py-12 text-center">
            <p className="text-lg font-bold text-white">Belum ada giveaway yang cocok</p>
            <p className="mt-2 text-sm text-slate-400">
              Coba pilih platform atau tipe lain untuk melihat giveaway yang tersedia.
            </p>
          </section>
        )}

        {status === "success" && filteredGiveaways.length > 0 && (
          <div className="grid gap-5 sm:grid-cols-2 xl:grid-cols-3">
            {filteredGiveaways.map((giveaway) => (
              <GiveawayCard key={giveaway.id} giveaway={giveaway} />
            ))}
          </div>
        )}

        <p className="mt-8 text-center text-xs leading-5 text-slate-500">
          Data giveaway disediakan oleh{" "}
          <a
            href="https://www.gamerpower.com"
            target="_blank"
            rel="noopener noreferrer"
            className="font-semibold text-emerald-300 underline underline-offset-4 hover:text-emerald-200"
          >
            GamerPower<span className="sr-only"> (buka di tab baru)</span>
          </a>
          . Ketersediaan dan batas klaim dapat berubah sewaktu-waktu; periksa syarat di halaman sumber.
        </p>
      </div>
    </main>
  );
}



// Serverless proxy GamerPower untuk production (Vercel).
// Frontend memanggil /api/gamerpower/giveaways?sort-by=date,
// fungsi ini meneruskannya ke https://www.gamerpower.com/api/giveaways?sort-by=date
export default async function handler(req, res) {
  const { path = [], ...query } = req.query;
  const sub = [].concat(path).join("/");
  const qs = new URLSearchParams(query).toString();
  const url = `https://www.gamerpower.com/api/${sub}${qs ? `?${qs}` : ""}`;

  try {
    const upstream = await fetch(url);
    const body = await upstream.text();
    res.setHeader(
      "Content-Type",
      upstream.headers.get("content-type") || "application/json"
    );
    res.setHeader("Cache-Control", "s-maxage=600, stale-while-revalidate=300");
    res.status(upstream.status).send(body);
  } catch {
    res.status(502).json({ message: "Gagal menghubungi GamerPower." });
  }
}