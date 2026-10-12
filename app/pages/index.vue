<script setup lang="ts">
import faviconUrl from "~/assets/icon.png";

type Shortcut = {
	id: string;
	name: string;
	url: string;
};

const STORAGE_KEY = "bloom-voting-shortcuts";
const THEME_STORAGE_KEY = "bloom-voting-theme";
type ThemeMode = "system" | "light" | "dark";

useSeoMeta({
	title: "Bloom Voting Manager",
	description: "A simple desk for your voting category shortcuts.",
});

useHead({
	link: [{ rel: "icon", type: "image/png", href: faviconUrl }],
});

const shortcuts = ref<Shortcut[]>([]);
const activeId = ref<string | null>(null);
const manageOpen = ref(false);
const iframeRef = ref<HTMLIFrameElement | null>(null);
const viewerStageRef = ref<HTMLElement | null>(null);
const isFullscreen = ref(false);
const themeMode = ref<ThemeMode>("system");
const prefersDarkMode = ref(false);
const spotifyPlayerOpen = ref(false);
const nameInput = ref("");
const urlInput = ref("");
const editingId = ref<string | null>(null);
const hasLoaded = ref(false);
const errorMessage = ref("");

const activeShortcut = computed(
	() => shortcuts.value.find((shortcut) => shortcut.id === activeId.value) ?? null,
);
const isEditing = computed(() => editingId.value !== null);
const resolvedTheme = computed(() =>
	themeMode.value === "system" ? (prefersDarkMode.value ? "dark" : "light") : themeMode.value,
);
const embeddingBlockedBySite = computed(() => {
	if (!activeShortcut.value) return false;
	const hostname = new URL(activeShortcut.value.url).hostname.toLowerCase();
	return (
		hostname === "x.com" ||
		hostname.endsWith(".x.com") ||
		hostname === "twitter.com" ||
		hostname.endsWith(".twitter.com")
	);
});
let systemThemeQuery: MediaQueryList | null = null;

function updateSystemTheme(event: MediaQueryListEvent) {
	prefersDarkMode.value = event.matches;
}

function updateFullscreenState() {
	isFullscreen.value = document.fullscreenElement === viewerStageRef.value;
}

onMounted(() => {
	try {
		const stored = localStorage.getItem(STORAGE_KEY);
		if (stored) {
			const parsed: unknown = JSON.parse(stored);
			if (Array.isArray(parsed)) {
				shortcuts.value = parsed.filter(isShortcut);
				activeId.value = shortcuts.value[0]?.id ?? null;
			}
		}
		const storedTheme = localStorage.getItem(THEME_STORAGE_KEY);
		if (storedTheme === "system" || storedTheme === "light" || storedTheme === "dark") {
			themeMode.value = storedTheme;
		}
	} catch {}
	systemThemeQuery = window.matchMedia("(prefers-color-scheme: dark)");
	prefersDarkMode.value = systemThemeQuery.matches;
	systemThemeQuery.addEventListener("change", updateSystemTheme);
	hasLoaded.value = true;
	document.addEventListener("fullscreenchange", updateFullscreenState);
});

onBeforeUnmount(() => {
	document.removeEventListener("fullscreenchange", updateFullscreenState);
	systemThemeQuery?.removeEventListener("change", updateSystemTheme);
});

watch(
	shortcuts,
	(value) => {
		if (hasLoaded.value) {
			try {
				localStorage.setItem(STORAGE_KEY, JSON.stringify(value));
			} catch {}
		}
	},
	{ deep: true },
);

watch(themeMode, (value) => {
	if (hasLoaded.value) {
		try {
			localStorage.setItem(THEME_STORAGE_KEY, value);
		} catch {}
	}
});

function isShortcut(value: unknown): value is Shortcut {
	if (!value || typeof value !== "object") return false;
	const candidate = value as Partial<Shortcut>;
	return (
		typeof candidate.id === "string" &&
		typeof candidate.name === "string" &&
		typeof candidate.url === "string" &&
		isWebUrl(candidate.url)
	);
}

function isWebUrl(value: string): boolean {
	try {
		const parsed = new URL(value);
		return parsed.protocol === "https:" || parsed.protocol === "http:";
	} catch {
		return false;
	}
}

function saveShortcut() {
	const name = nameInput.value.trim();
	let url = urlInput.value.trim();
	if (!name || !url) {
		errorMessage.value = "Add a category name and URL.";
		return;
	}

	if (!/^https?:\/\//i.test(url)) url = `https://${url}`;
	if (!isWebUrl(url)) {
		errorMessage.value = "Enter a valid website URL.";
		return;
	}

	if (editingId.value) {
		const shortcut = shortcuts.value.find((item) => item.id === editingId.value);
		if (shortcut) {
			shortcut.name = name;
			shortcut.url = url;
		}
	} else {
		const shortcut = { id: crypto.randomUUID(), name, url };
		shortcuts.value.push(shortcut);
		activeId.value = shortcut.id;
	}

	nameInput.value = "";
	urlInput.value = "";
	editingId.value = null;
	errorMessage.value = "";
	manageOpen.value = false;
}

function editShortcut(shortcut: Shortcut) {
	manageOpen.value = true;
	editingId.value = shortcut.id;
	nameInput.value = shortcut.name;
	urlInput.value = shortcut.url;
	errorMessage.value = "";
}

function cancelEdit() {
	editingId.value = null;
	nameInput.value = "";
	urlInput.value = "";
	errorMessage.value = "";
}

function removeShortcut(id: string) {
	const index = shortcuts.value.findIndex((shortcut) => shortcut.id === id);
	if (index < 0) return;
	shortcuts.value.splice(index, 1);
	if (activeId.value === id) {
		activeId.value = shortcuts.value[Math.min(index, shortcuts.value.length - 1)]?.id ?? null;
	}
	if (editingId.value === id) cancelEdit();
}

function reloadFrame() {
	if (activeShortcut.value && iframeRef.value) {
		iframeRef.value.src = activeShortcut.value.url;
	}
}

async function toggleFrameFullscreen() {
	if (!viewerStageRef.value) return;

	try {
		if (document.fullscreenElement === viewerStageRef.value) {
			await document.exitFullscreen();
		} else {
			await viewerStageRef.value.requestFullscreen();
		}
	} catch {
		isFullscreen.value = false;
	}
}
</script>

<template>
	<div class="app-shell" :data-theme="resolvedTheme">
		<header class="topbar">
			<a class="brand" href="/" aria-label="Bloom Voting home">
				<span class="brand-name">bloom<span>.</span></span>
			</a>
			<nav class="nav-controls" aria-label="Voting controls">
				<label class="category-picker" for="category-select">
					<span>LINKS</span>
					<select
						id="category-select"
						v-model="activeId"
						:disabled="shortcuts.length === 0"
					>
						<option :value="null" disabled>Choose a link</option>
						<option
							v-for="shortcut in shortcuts"
							:key="shortcut.id"
							:value="shortcut.id"
						>
							{{ shortcut.name }}
						</option>
					</select>
				</label>
				<button
					class="manage-toggle"
					type="button"
					:aria-expanded="manageOpen"
					aria-controls="manage-panel"
					@click="manageOpen = !manageOpen"
				>
					{{ manageOpen ? "Close" : "Manage links" }}
					<span class="manage-count">{{ shortcuts.length }}</span>
				</button>
				<a
					v-if="activeShortcut"
					class="open-link"
					:href="activeShortcut.url"
					target="_blank"
					rel="noopener noreferrer"
					aria-label="Open current category in a new tab"
					title="Open in new tab"
				>
					<span class="open-link-label">Open in new tab</span>
					<span aria-hidden="true">↗</span>
				</a>
			</nav>
			<div class="spotify-menu" :class="{ 'is-open': spotifyPlayerOpen }">
				<button
					class="spotify-toggle"
					type="button"
					:aria-expanded="spotifyPlayerOpen"
					aria-controls="spotify-player"
					@click="spotifyPlayerOpen = !spotifyPlayerOpen"
				>
					<span>Spotify</span>
					<span>{{ spotifyPlayerOpen ? "Close" : "Music" }}</span>
				</button>
				<div class="spotify-player-window" v-show="spotifyPlayerOpen">
					<iframe
						id="spotify-player"
						class="spotify-player"
						src="https://open.spotify.com/embed/playlist/1eoVOqdyxzDkOtkGnVf58V?utm_source=generator&theme=0"
						title="Spotify playlist background music"
						allow="
							autoplay;
							clipboard-write;
							encrypted-media;
							fullscreen;
							picture-in-picture;
						"
						loading="eager"
					></iframe>
				</div>
			</div>
		</header>

		<main class="page-content">
			<section
				v-if="manageOpen"
				id="manage-panel"
				class="manage-panel"
				aria-label="Manage links"
			>
				<div class="manage-categories">
					<div class="manage-heading">
						<h2>Saved links</h2>
						<span>Saved in this browser</span>
					</div>
					<ul v-if="shortcuts.length" class="manage-list">
						<li v-for="shortcut in shortcuts" :key="shortcut.id" class="manage-item">
							<button
								class="manage-item-select"
								type="button"
								@click="activeId = shortcut.id"
							>
								<span class="manage-item-name">{{ shortcut.name }}</span>
								<span class="manage-item-url">{{ shortcut.url }}</span>
							</button>
							<div class="manage-item-actions">
								<button
									class="text-action"
									type="button"
									:aria-label="`Edit ${shortcut.name}`"
									@click="editShortcut(shortcut)"
								>
									Edit
								</button>
								<button
									class="text-action remove-action"
									type="button"
									:aria-label="`Remove ${shortcut.name}`"
									@click="removeShortcut(shortcut.id)"
								>
									Remove
								</button>
							</div>
						</li>
					</ul>
					<p v-else class="manage-empty">No links yet.</p>
				</div>

				<form class="shortcut-form" @submit.prevent="saveShortcut">
					<div class="form-heading">
						<h2>{{ isEditing ? "Edit link" : "Add a link" }}</h2>
						<button
							v-if="isEditing"
							class="cancel-edit"
							type="button"
							@click="cancelEdit"
						>
							Cancel
						</button>
					</div>
					<div class="form-fields">
						<label class="form-field" for="category-name">
							<span>Name</span>
							<input
								id="category-name"
								v-model="nameInput"
								class="text-input"
								type="text"
								maxlength="48"
								placeholder="e.g. Favorite character"
								autocomplete="off"
							/>
						</label>
						<label class="form-field" for="category-url">
							<span>URL</span>
							<input
								id="category-url"
								v-model="urlInput"
								class="text-input"
								type="text"
								inputmode="url"
								placeholder="https://…"
								autocomplete="url"
							/>
						</label>
						<button class="save-button" type="submit">
							{{ isEditing ? "Save" : "Add link" }} <span aria-hidden="true">→</span>
						</button>
					</div>
					<p v-if="errorMessage" class="form-error" role="alert">{{ errorMessage }}</p>
				</form>
			</section>

			<section class="viewer" aria-label="Website viewer">
				<div ref="viewerStageRef" class="viewer-stage">
					<div class="viewer-bar">
						<span class="viewer-kicker">{{
							activeShortcut?.name ?? "VOTING WEBSITE"
						}}</span>
						<span v-if="activeShortcut" class="viewer-url">{{
							activeShortcut.url
						}}</span>
						<div class="viewer-actions">
							<label class="theme-picker">
								<span class="visually-hidden">Color theme</span>
								<select
									v-model="themeMode"
									aria-label="Color theme"
									title="Color theme"
								>
									<option value="system">System</option>
									<option value="light">Light</option>
									<option value="dark">Dark</option>
								</select>
							</label>
							<button
								class="viewer-action"
								type="button"
								:disabled="!activeShortcut || embeddingBlockedBySite"
								aria-label="Reload embedded page"
								title="Reload embedded page"
								@click="reloadFrame"
							>
								<span aria-hidden="true">↻</span
								><span class="viewer-action-label">Reload</span>
							</button>
							<button
								class="viewer-action"
								type="button"
								:disabled="!activeShortcut || embeddingBlockedBySite"
								:aria-label="
									isFullscreen
										? 'Exit full screen'
										: 'Show embedded page full screen'
								"
								:aria-pressed="isFullscreen"
								:title="isFullscreen ? 'Exit full screen' : 'Show full screen'"
								@click="toggleFrameFullscreen"
							>
								<span aria-hidden="true">{{ isFullscreen ? "×" : "⛶" }}</span
								><span class="viewer-action-label">{{
									isFullscreen ? "Exit full screen" : "Full screen"
								}}</span>
							</button>
						</div>
					</div>
					<div
						v-if="isFullscreen"
						class="fullscreen-menu"
						role="toolbar"
						aria-label="Fullscreen controls"
					>
						<label class="fullscreen-category">
							<span class="visually-hidden">Saved links</span>
							<select v-model="activeId" aria-label="Choose a link">
								<option
									v-for="shortcut in shortcuts"
									:key="shortcut.id"
									:value="shortcut.id"
								>
									{{ shortcut.name }}
								</option>
							</select>
						</label>
						<label class="fullscreen-theme">
							<span class="visually-hidden">Color theme</span>
							<select v-model="themeMode" aria-label="Color theme">
								<option value="system">System</option>
								<option value="light">Light</option>
								<option value="dark">Dark</option>
							</select>
						</label>
						<button
							type="button"
							aria-label="Reload embedded page"
							title="Reload embedded page"
							@click="reloadFrame"
						>
							↻
						</button>
						<button
							type="button"
							class="fullscreen-exit"
							@click="toggleFrameFullscreen"
						>
							Back to page
						</button>
					</div>
					<iframe
						v-if="activeShortcut && !embeddingBlockedBySite"
						ref="iframeRef"
						:key="activeShortcut.id + activeShortcut.url"
						class="site-frame"
						:src="activeShortcut.url"
						:title="`${activeShortcut.name} voting page`"
						:style="{ colorScheme: resolvedTheme }"
						referrerpolicy="strict-origin-when-cross-origin"
						allow="fullscreen; clipboard-write"
						allowfullscreen
					></iframe>
					<div v-else-if="activeShortcut" class="embed-blocked" role="status">
						<p class="embed-blocked-title">X doesn’t allow this page to be embedded.</p>
						<p>Open the link in a new tab to continue.</p>
						<a
							class="empty-action"
							:href="activeShortcut.url"
							target="_blank"
							rel="noopener noreferrer"
						>
							Open in new tab <span aria-hidden="true">↗</span>
						</a>
					</div>
					<div v-else class="viewer-empty">
						<p>No category selected</p>
						<button type="button" class="empty-action" @click="manageOpen = true">
							Add a voting link
						</button>
					</div>
				</div>
			</section>
		</main>
	</div>
</template>
