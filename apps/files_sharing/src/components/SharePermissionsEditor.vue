<template>
	<span>
		<template v-if="!showCustomPermissionsForm">
			<ActionRadio :checked="isPermission(bundledPermissions.READ_ONLY)"
				:value="bundledPermissions.READ_ONLY"
				:name="randomFormName"
				:disabled="saving"
				@change="setPermissions(bundledPermissions.READ_ONLY)">
				{{ t('files_sharing', 'Read only') }}
			</ActionRadio>

			<!-- folder -->
			<template v-if="isFolder && fileHasCreatePermission && config.isPublicUploadEnabled">
				<ActionRadio :checked="isPermission(bundledPermissions.UPLOAD_AND_EDIT)"
					:value="bundledPermissions.UPLOAD_AND_EDIT"
					:disabled="saving"
					:name="randomFormName"
					@change="setPermissions(bundledPermissions.UPLOAD_AND_EDIT)">
					{{ t('files_sharing', 'Allow upload and editing') }}
				</ActionRadio>
				<ActionRadio :checked="isPermission(bundledPermissions.FILE_DROP)"
					:value="bundledPermissions.FILE_DROP"
					:disabled="saving"
					:name="randomFormName"
					class="sharing-entry__action--public-upload"
					@change="setPermissions(bundledPermissions.FILE_DROP)">
					{{ t('files_sharing', 'File drop (upload only)') }}
				</ActionRadio>
			</template>

			<!-- file -->
			<ActionRadio v-if="!isFolder"
				:checked="isPermission(bundledPermissions.EDIT_ONLY)"
				:value="bundledPermissions.EDIT_ONLY"
				:name="randomFormName"
				:disabled="saving"
				@change="setPermissions(bundledPermissions.EDIT_ONLY)">
				{{ t('files_sharing', 'Allow editing') }}
			</ActionRadio>

			<!-- custom permissions button -->
			<ActionButton :title="t('files_sharing', 'Manual permissions')"
				@click="showCustomPermissionsForm = true">
				<template #icon>
					<DotsHorizontal />
				</template>
				{{ permissionsSummary }}
			</ActionButton>

		</template>
		<template v-else>
			<!-- custom permissions -->
			<ActionButton @click="showCustomPermissionsForm = false">
				<template #icon>
					<ChevronLeft />
				</template>
				{{ t('files_sharing', 'Manual permissions') }}
			</ActionButton>

			<ActionCheckbox :checked="hasPermission(atomicPermissions.CREATE)"
				:disabled="saving"
				@update:checked="togglePermissions(atomicPermissions.CREATE)">
				{{ t('files_sharing', 'Create') }}
			</ActionCheckbox>
			<ActionCheckbox :checked="hasPermission(atomicPermissions.READ)"
				:disabled="saving"
				@update:checked="togglePermissions(atomicPermissions.READ)">
				{{ t('files_sharing', 'Read') }}
			</ActionCheckbox>
			<ActionCheckbox :checked="hasPermission(atomicPermissions.UPDATE)"
				:disabled="saving"
				@update:checked="togglePermissions(atomicPermissions.UPDATE)">
				{{ t('files_sharing', 'Update') }}
			</ActionCheckbox>
			<ActionCheckbox :checked="hasPermission(atomicPermissions.DELETE)"
				:disabled="saving"
				@update:checked="togglePermissions(atomicPermissions.DELETE)">
				{{ t('files_sharing', 'Delete') }}
			</ActionCheckbox>
		</template>
	</span>
</template>

<script>
import ActionButton from '@nextcloud/vue/dist/Components/ActionButton'
import ActionRadio from '@nextcloud/vue/dist/Components/ActionRadio'
import ActionCheckbox from '@nextcloud/vue/dist/Components/ActionCheckbox'

import SharesMixin from '../mixins/SharesMixin'

import DotsHorizontal from 'vue-material-design-icons/DotsHorizontal.vue'
import ChevronLeft from 'vue-material-design-icons/ChevronLeft.vue'

export default {
	name: 'SharePermissionsEditor',

	components: {
		ActionButton,
		ActionCheckbox,
		ActionRadio,
		DotsHorizontal,
		ChevronLeft,
	},

	mixins: [SharesMixin],

	data() {
		return {
			randomFormName: Math.random().toString(27).substr(2),

			showCustomPermissionsForm: false,

			atomicPermissions: {
				CREATE: OC.PERMISSION_CREATE,
				READ: OC.PERMISSION_READ,
				UPDATE: OC.PERMISSION_UPDATE,
				DELETE: OC.PERMISSION_DELETE,
			},

			bundledPermissions: {
				READ_ONLY: OC.PERMISSION_READ,
				UPLOAD_AND_EDIT: OC.PERMISSION_UPDATE | OC.PERMISSION_CREATE | OC.PERMISSION_READ | OC.PERMISSION_DELETE,
				FILE_DROP: OC.PERMISSION_CREATE,
				EDIT_ONLY: OC.PERMISSION_UPDATE | OC.PERMISSION_CREATE | OC.PERMISSION_READ | OC.PERMISSION_DELETE,
			},
		}
	},

	computed: {
		/**
		 * Return the current share permissions
		 * We always ignore the SHARE permission as this is used for the
		 * federated sharing.
		 *
		 * @return {number}
		 */
		permissionsWithoutSharePerm() {
			return this.share.permissions & ~OC.PERMISSION_SHARE
		},

		/**
		 * Return a summary of checked permissions.
		 */
		permissionsSummary() {
			return Object.values(this.atomicPermissions)
				.filter(permission => this.hasPermission(permission))
				.map(permission => {
					switch (permission) {
					case this.atomicPermissions.CREATE:
						return this.t('files_sharing', 'Create')
					case this.atomicPermissions.READ:
						return this.t('files_sharing', 'Read')
					case this.atomicPermissions.UPDATE:
						return this.t('files_sharing', 'Update')
					case this.atomicPermissions.DELETE:
						return this.t('files_sharing', 'Delete')
					default:
						return ''
					}
				})
				.join(', ')
		},

		/**
		 * Return wether the share's permission is a bundle
		 */
		isBundledPermissions() {
			return Object.values(this.bundledPermissions)
				.map(bundle => this.isPermission(bundle))
				.filter(isBundle => isBundle)
				.length > 0
		},

		/**
		 * Is the current share a folder ?
		 * TODO: move to a proper FileInfo model?
		 *
		 * @return {boolean}
		 */
		isFolder() {
			return this.fileInfo.type === 'dir'
		},

		/**
		 * Does the current file/folder have create permissions
		 * TODO: move to a proper FileInfo model?
		 *
		 * @return {boolean}
		 */
		fileHasCreatePermission() {
			return !!(this.fileInfo.permissions & OC.PERMISSION_CREATE)
		},
	},

	mounted() {
		this.showCustomPermissionsForm = !this.isBundledPermissions
	},

	methods: {
		/**
		 * Return wether the share has the exact given permissions.
		 *
		 * @param {string} permissions - the permissions to check.
		 */
		isPermission(permissions) {
			return (this.share.permissions & ~OC.PERMISSION_SHARE) === permissions
		},

		/**
		 * Return wether the share has the given permissions.
		 *
		 * @param {string} permissions - the permissions to check.
		 */
		hasPermission(permissions) {
			return this.share.permissions !== 0 && (this.share.permissions & permissions) === permissions
		},

		/**
		 * Add some permissions to the share.
		 *
		 * @param {string} permissions - the permissions to add.
		 */
		addPermission(permissions) {
			this.share.permissions |= permissions
			this.queueUpdate('permissions')
		},

		/**
		 * Remove some permissions to the share.
		 *
		 * @param {string} permissions - the permissions to remove.
		 */
		removePermission(permissions) {
			this.share.permissions &= ~permissions
			this.queueUpdate('permissions')
		},

		/**
		 * Set the share permissions to the given permissions.
		 *
		 * @param {string} permissions - the permissions to set.
		 */
		setPermissions(permissions) {
			this.share.permissions = parseInt(permissions, 10)
			this.queueUpdate('permissions')
		},

		/**
		 * Toggle a given permission.
		 *
		 * @param {string} permission - the permission to toggle.
		 */
		togglePermissions(permission) {
			permission = parseInt(permission, 10)

			if (this.hasPermission(permission)) {
				this.removePermission(permission)
			} else {
				this.addPermission(permission)
			}
		},
	},
}
</script>

<style>

</style>
