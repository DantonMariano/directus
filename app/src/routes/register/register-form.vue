<script setup lang="ts">
import api, { RequestError } from '@/api';
import { login } from '@/auth';
import { translateAPIError } from '@/lang';
import { useServerStore } from '@/stores/server';
import { useUserStore } from '@/stores/user';
import { ErrorCode } from '@directus/errors';
import { computed, ref } from 'vue';
import { useI18n } from 'vue-i18n';
import { useRouter } from 'vue-router';

type Credentials = {
	email: string;
	password: string;
};

const { t } = useI18n();
const router = useRouter();
const serverStore = useServerStore();
const userStore = useUserStore();
const isLoading = ref(false);
const email = ref<string | null>(null);
const password = ref<string | null>(null);
const error = ref<RequestError | string | null>(null);

const emit = defineEmits<{
	wasSuccessful: [boolean];
}>();

const requiresEmailVerification = computed(() => serverStore.info.project?.public_registration_verify_email);

const errorFormatted = computed(() => {
	// Show "Wrong username or password" for wrongly formatted emails as well
	if (error.value === ErrorCode.InvalidPayload) {
		return translateAPIError(ErrorCode.InvalidCredentials);
	}

	if (error.value) {
		return translateAPIError(error.value);
	}

	return null;
});

async function onSubmit() {
	// Simple RegEx, not for validation, but to prevent unnecessary login requests when the value is clearly invalid
	const emailRegex = /^\S+@\S+$/;

	if (email.value === null || !emailRegex.test(email.value) || password.value === null) {
		error.value = ErrorCode.InvalidPayload;
		return;
	}

	try {
		isLoading.value = true;

		const credentials: Credentials = {
			email: email.value,
			password: password.value,
		};

		await api.post('/users/register', credentials);

		// If email verification is not required, automatically log in the user
		if (!requiresEmailVerification.value) {
			await login({ credentials });
			const currentUser = userStore.currentUser;

			if (currentUser && 'id' in currentUser) {
				router.push(`/users/${currentUser.id}`);
			} else {
				router.push('/login');
			}
		} else {
			// If email verification is required, show success message
			emit('wasSuccessful', true);
		}
	} catch (err: any) {
		error.value = err.errors?.[0]?.extensions?.code || err;
		emit('wasSuccessful', false);
	} finally {
		isLoading.value = false;
	}
}
</script>

<template>
	<form novalidate @submit.prevent="onSubmit">
		<v-input
			v-model="email"
			autofocus
			autocomplete="username"
			type="email"
			:placeholder="t('email')"
			:disabled="isLoading"
		/>
		<interface-system-input-password :value="password" :disabled="isLoading" @input="password = $event" />

		<v-notice v-if="error" type="warning">
			{{ errorFormatted }}
		</v-notice>
		<div class="buttons">
			<v-button type="submit" :loading="isLoading" :disabled="isLoading" large>{{ t('register') }}</v-button>
		</div>
	</form>
</template>

<style lang="scss" scoped>
.v-input,
.v-notice {
	margin-block-end: 20px;
}

.buttons {
	display: flex;
	align-items: center;
	justify-content: space-between;
}

.forgot-password {
	color: var(--theme--foreground-subdued);
	transition: color var(--fast) var(--transition);

	&:hover {
		color: var(--theme--foreground);
	}
}
</style>
