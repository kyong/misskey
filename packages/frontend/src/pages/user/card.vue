<!--
SPDX-FileCopyrightText: syuilo and misskey-project
SPDX-License-Identifier: AGPL-3.0-only
-->

<template>
<MkSpacer :contentMax="narrow ? 800 : 1100">
	<div ref="rootEl" class="ftskorzw" :class="{ wide: !narrow }" style="container-type: inline-size;">
		<div class="main _gaps">
			<div class="card _gaps">
				<div :key="user.id" ref="captureElement" class="main _panel">
					<div class="card-container" :style="style">
					</div>
					<MkAvatar class="avatar" :user="user"/>
					<div class="title">
						<MkUserName :user="user" :nowrap="false" class="name"/>
						<div class="bottom">
							<span class="username"><MkAcct :user="user" :detail="true"/></span>
							<span v-if="user.isAdmin" :title="i18n.ts.isAdmin" style="color: var(--MI_THEME-badge);"><i class="ti ti-shield"></i></span>
							<span v-if="user.isLocked" :title="i18n.ts.isLocked"><i class="ti ti-lock"></i></span>
							<span v-if="user.isBot" :title="i18n.ts.isBot"><i class="ti ti-robot"></i></span>
						</div>
					</div>
					<div class="description">
						<MkOmit>
							<Mfm v-if="user.description" :text="user.description" :isNote="false" :author="user"/>
							<p v-else class="empty">{{ i18n.ts.noAccountDescription }}</p>
						</MkOmit>
					</div>

					<div v-if="user.roles.length > 0" class="roles">
						<span v-for="role in user.roles" :key="role.id" v-tooltip="role.description" class="role" :style="{ '--color': role.color }">
							<img v-if="role.iconUrl" style="height: 1.3em; vertical-align: -22%;" :src="role.iconUrl"/>
							{{ role.name }}
						</span>
					</div>
				</div>
			</div>
			<div class="share">
				<MkButton inline primary :label="'download'" @click="captureAndShare"><i class="ti ti-download"></i>Download</MkButton>
			</div>
		</div>
	</div>
</MkSpacer>
</template>

<script lang="ts" setup>
import { computed, onMounted, onUnmounted, watch, ref } from 'vue';
import * as Misskey from 'misskey-js';
import { getScrollPosition } from '@@/js/scroll.js';
import html2canvas from 'html2canvas';
// import * as os from '@/os.js';
import { i18n } from '@/i18n.js';
import { defaultStore } from '@/store.js';
import { confetti } from '@/scripts/confetti.js';
import { misskeyApi } from '@/scripts/misskey-api.js';
import { getStaticImageUrl } from '@/scripts/media-proxy.js';
import MkButton from '@/components/MkButton.vue';

const props = withDefaults(defineProps<{
	user: Misskey.entities.UserDetailed;
	/** Test only; MkNotes currently causes problems in vitest */
	disableNotes: boolean;
}>(), {
	disableNotes: false,
});

const user = ref(props.user);
const parallaxAnimationId = ref<null | number>(null);
const narrow = ref<null | boolean>(null);
const rootEl = ref<null | HTMLElement>(null);
const bannerEl = ref<null | HTMLElement>(null);
const moderationNote = ref(props.user.moderationNote);
const captureElement = ref<HTMLElement | null>(null);

watch(moderationNote, async () => {
	await misskeyApi('admin/update-user-note', { userId: props.user.id, text: moderationNote.value });
});

const style = computed(() => {
	if (props.user.bannerUrl == null) return {};
	if (defaultStore.state.disableShowingAnimatedImages) {
		return {
			backgroundImage: `url(${ getStaticImageUrl(props.user.bannerUrl) })`,
		};
	} else {
		return {
			backgroundImage: `url(${ props.user.bannerUrl })`,
		};
	};
});

function parallaxLoop() {
	parallaxAnimationId.value = window.requestAnimationFrame(parallaxLoop);
	parallax();
}

function parallax() {
	const banner = bannerEl.value;
	if (banner == null) return;

	const top = getScrollPosition(rootEl.value);

	if (top < 0) return;

	const z = 1.75; // 奥行き(小さいほど奥)
	const pos = -(top / z);
	banner.style.backgroundPosition = `center calc(50% - ${pos}px)`;
}

onMounted(() => {
	window.requestAnimationFrame(parallaxLoop);
	narrow.value = rootEl.value!.clientWidth < 1000;

	if (props.user.birthday) {
		const m = new Date().getMonth() + 1;
		const d = new Date().getDate();
		const bm = parseInt(props.user.birthday.split('-')[1]);
		const bd = parseInt(props.user.birthday.split('-')[2]);
		if (m === bm && d === bd) {
			confetti({
				duration: 1000 * 4,
			});
		}
	}
});

const captureAndShare = async () => {
	if (!captureElement.value) return;

	try {
		// willReadFrequently オプションを追加
		const canvas = await html2canvas(captureElement.value, {
			willReadFrequently: true,
			allowTaint: true,
			useCORS: true,
		});

		// Canvas を Blob に変換
		canvas.toBlob((blob) => {
			if (blob) {
				const url = URL.createObjectURL(blob);
				const a = document.createElement('a');
				a.href = url;
				a.download = 'screenshot.png';
				a.click();
				URL.revokeObjectURL(url);
			}
		}, 'image/png');
	} catch (error) {
		console.error('キャプチャエラー:', error);
	}
};

onUnmounted(() => {
	if (parallaxAnimationId.value) {
		window.cancelAnimationFrame(parallaxAnimationId.value);
	}
});
</script>

<style lang="scss" scoped>
.ftskorzw {

	> .main {

		> .punished {
			font-size: 0.8em;
			padding: 16px;
		}

		> .card {

			> .main {
				position: relative;
				overflow: clip;
				width: 600px;
				height: 360px;

				> .card-container {
					width: 100%;
					height: 100%;
					display: flex;
					flex-direction: column;
					overflow: clip;
					background-color: #4c5e6d;
					background-size: cover;
					background-position: center;
				}

				> .avatar {
					display: block;
					position: absolute;
					top: 80px;
					left: 20px;
					z-index: 2;
					width: 200px;
					height: 200px;
					box-shadow: 1px 1px 3px var(--MI_THEME-shadow);
				}

				> .title {
					display: block;
					position: absolute;
					top: 28px;
					left: 240px;
					background-color: var(--MI_THEME-bg);
					border-radius: 4px;
					padding: 8px 12px;
					opacity: 0.8;
					box-shadow: 1px 1px 3px var(--MI_THEME-shadow);

					> .username	{
						font-size: 1.3em;
						color: var(--MI_THEME-fg);
					}
				}

				> .description {
					display: block;
					position: absolute;
					top: 100px;
					left: 240px;
					font-size: 0.95em;
					background-color: var(--MI_THEME-bg);
					border-radius: 4px;
					width: 310px;
					height: 140px;
					padding: 8px 12px;
					overflow: hidden;
					text-overflow: ellipsis;
					white-space: nowrap;
					text-align: left;
					color: var(--MI_THEME-fg);
					opacity: 0.8;
					box-shadow: 1px 1px 3px var(--MI_THEME-shadow);

				}

				> .roles {
					display: block;
					position: absolute;
					top: 270px;
					left: 240px;
					font-size: 0.95em;
					background-color: var(--MI_THEME-bg);
					border-radius: 4px;
					width: 310px;
					height: 45px;
					padding: 8px 12px;
					overflow: hidden;
					text-overflow: ellipsis;
					white-space: nowrap;
					text-align: left;
					opacity: 0.8;
					box-shadow: 1px 1px 3px var(--MI_THEME-shadow);

					> .role {
						border: solid 1px var(--color, var(--MI_THEME-divider));
						border-radius: 999px;
						margin-right: 4px;
						padding: 3px 8px;
					}
				}
			}
		}

		> .share{
			width: 100%;
			height: 300px;
		}
	}

	&.wide {
		display: flex;
		width: 100%;

		> .main {
			width: 100%;
			min-width: 0;
		}

		> .sub {
			max-width: 350px;
			min-width: 350px;
			margin-left: var(--MI-margin);
		}
	}
}

</style>

<style lang="scss" module>
.tl {
	background: var(--MI_THEME-bg);
	border-radius: var(--MI-radius);
	overflow: clip;
}

.verifiedLink {
	margin-left: 4px;
	color: var(--MI_THEME-success);
}
</style>
