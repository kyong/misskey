<!--
SPDX-FileCopyrightText: syuilo and misskey-project
SPDX-License-Identifier: AGPL-3.0-only
-->

<template>
<div class="_gaps">
	<div v-if="rooms.length > 0" class="_gaps_s">
		<XRoom v-for="room in rooms" :key="room.id" :room="room"/>
	</div>
	<div v-if="!fetching && rooms.length == 0" class="_fullinfo">
		<div>{{ i18n.ts._chat.noRooms }}</div>
	</div>
	<MkLoading v-if="fetching"/>
</div>
</template>

<script lang="ts" setup>
import { computed, onMounted, ref } from 'vue';
import * as Misskey from 'misskey-js';
import XRoom from './XRoom.vue';
import MkButton from '@/components/MkButton.vue';
import { i18n } from '@/i18n.js';
import { misskeyApi } from '@/utility/misskey-api.js';
import { ensureSignin } from '@/i.js';
import { useRouter } from '@/router.js';
import * as os from '@/os.js';

const $i = ensureSignin();

const router = useRouter();

const fetching = ref(true);
const rooms = ref<Misskey.entities.ChatRoom[]>([]);

async function fetchRooms() {
	fetching.value = true;

	const res = await misskeyApi('chat/rooms/owned', {
	});

	rooms.value = res;

	fetching.value = false;
}

onMounted(() => {
	fetchRooms();
});
</script>

<style lang="scss" module>

</style>
