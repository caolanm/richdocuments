<!--
  - @copyright Copyright (c) 2019 Julius Härtl <jus@bitgrid.net>
  -
  - @author Julius Härtl <jus@bitgrid.net>
  -
  - @license GNU AGPL version 3 or any later version
  -
  - This program is free software: you can redistribute it and/or modify
  - it under the terms of the GNU Affero General Public License as
  - published by the Free Software Foundation, either version 3 of the
  - License, or (at your option) any later version.
  -
  - This program is distributed in the hope that it will be useful,
  - but WITHOUT ANY WARRANTY; without even the implied warranty of
  - MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
  - GNU Affero General Public License for more details.
  -
  - You should have received a copy of the GNU Affero General Public License
  - along with this program. If not, see <http://www.gnu.org/licenses/>.
  -
  -->

<template>
	<div class="office-viewer">
		<div v-if="showLoadingIndicator"
			class="office-viewer__loading-overlay"
			:class="{ debug: debug }">
			<NcEmptyContent v-if="!error" :title="t('richdocuments', 'Loading {filename} …', { filename: basename }, 1, {escape: false})">
				<template #icon>
					<NcLoadingIcon />
				</template>
				<template #action>
					<NcButton @click="close">
						{{ t('richdocuments', 'Cancel') }}
					</NcButton>
				</template>
			</NcEmptyContent>
			<NcEmptyContent v-else :title="t('richdocuments', 'Document loading failed')" :description="errorMessage">
				<template #icon>
					<AlertOctagonOutline />
				</template>
				<template #action>
					<NcButton @click="close">
						{{ t('richdocuments', 'Close') }}
					</NcButton>
				</template>
			</NcEmptyContent>
		</div>
		<form ref="form"
			:target="iframeId"
			:action="formData.action"
			method="post">
			<input name="access_token" :value="formData.accessToken" type="hidden">
			<input name="access_token_ttl" :value="formData.accessTokenTTL" type="hidden">
			<input name="ui_defaults" :value="formData.uiDefaults" type="hidden">
			<input name="css_variables" :value="formData.cssVariables" type="hidden">
			<input name="theme" :value="formData.theme" type="hidden">
			<input name="buy_product" value="https://nextcloud.com/pricing" type="hidden">
		</form>
		<iframe :id="iframeId"
			ref="documentFrame"
			:name="iframeId"
			data-cy="coolframe"
			scrolling="no"
			allowfullscreen
			class="office-viewer__iframe"
			:style="{visibility: showIframe ? 'visible' : 'hidden' }"
			:src="iframeSrc" />

		<ZoteroHint :show.sync="showZotero" @submit="reload" />
	</div>
</template>

<script>
import { NcButton, NcEmptyContent, NcLoadingIcon } from '@nextcloud/vue'
import AlertOctagonOutline from 'vue-material-design-icons/AlertOctagonOutline.vue'
import { loadState } from '@nextcloud/initial-state'

import ZoteroHint from '../components/Modal/ZoteroHint.vue'
import { basename, dirname } from 'path'
import { getRandomId } from '../helpers/index.js'
import {
	getNextcloudUrl,
	getWopiUrl,
} from '../helpers/url.js'
import PostMessageService from '../services/postMessage.tsx'
import FilesAppIntegration from './FilesAppIntegration.js'
import { LOADING_ERROR, checkCollaboraConfiguration, checkProxyStatus } from '../services/collabora.js'
import { enableScrollLock, disableScrollLock } from '../helpers/safariFixer.js'
import axios from '@nextcloud/axios'
import {
	generateUrl,
	imagePath,
} from '@nextcloud/router'
import { getCapabilities } from '@nextcloud/capabilities'
import {
	generateCSSVarTokens,
	getCollaboraTheme,
	getUIDefaults,
} from '../helpers/coolParameters.js'
import Config from '../services/config.tsx'
import openLocal from '../mixins/openLocal.js'
import pickLink from '../mixins/pickLink.js'
import saveAs from '../mixins/saveAs.js'
import uiMention from '../mixins/uiMention.js'
import version from '../mixins/version.js'

const FRAME_DOCUMENT = 'FRAME_DOCUMENT'

const LOADING_STATE = {
	LOADING: 0,
	FRAME_READY: 1,
	DOCUMENT_READY: 2,
	FAILED: -1,
}

export default {
	name: 'Office',
	components: {
		AlertOctagonOutline,
		NcButton,
		NcEmptyContent,
		NcLoadingIcon,
		ZoteroHint,
	},
	mixins: [
		openLocal, pickLink, saveAs, uiMention, version,
	],
	props: {
		filename: {
			type: String,
			default: null,
		},
		fileid: {
			type: Number,
			default: null,
		},
		hasPreview: {
			type: Boolean,
			required: false,
			default: () => false,
		},
		source: {
			type: String,
			default: null,
		},
	},
	data() {
		return {
			postMessage: null,
			iframeId: 'collaboraframe_' + getRandomId(),
			iframeSrc: null,
			loading: LOADING_STATE.LOADING,
			loadingTimeout: null,
			error: null,
			views: [],

			showLinkPicker: false,
			showZotero: false,
			modified: false,

			formData: {
				action: null,
				accessToken: null,
				accessTokenTTL: null,
				uiDefaults: getUIDefaults(),
				cssVariables: generateCSSVarTokens(),
				theme: getCollaboraTheme(),
			},
		}
	},
	computed: {
		showIframe() {
			return this.loading >= LOADING_STATE.FRAME_READY || this.debug
		},
		showLoadingIndicator() {
			return this.loading < LOADING_STATE.FRAME_READY
		},
		errorMessage() {
			switch (parseInt(this.error)) {
			case LOADING_ERROR.COLLABORA_UNCONFIGURED:
				return t('richdocuments', '{productName} is not configured', { productName: loadState('richdocuments', 'productName', 'Nextcloud Office') })
			case LOADING_ERROR.PROXY_FAILED:
				return t('richdocuments', 'Starting the built-in CODE server failed')
			default:
				return this.error
			}
		},
		debug() {
			return !!window.TESTING
		},
		isPublic() {
			return document.getElementById('isPublic')?.value === '1'
		},
		shareToken() {
			return document.getElementById('sharingToken')?.value
		},
	},
	async mounted() {
		this.postMessage = new PostMessageService({
			FRAME_DOCUMENT: () => document.getElementById(this.iframeId).contentWindow,
		})
		try {
			await checkCollaboraConfiguration()
			await checkProxyStatus()
		} catch (e) {
			this.error = e.message
			this.loading = LOADING_STATE.FAILED
			return
		}

		if (this.fileid) {
			const fileList = OCA?.Files?.App?.getCurrentFileList?.()
			FilesAppIntegration.init({
				fileName: basename(this.filename),
				fileId: this.fileid,
				filePath: dirname(this.filename),
				fileList,
				fileModel: fileList?.getModelForFile(basename(this.filename)),
				sendPostMessage: (msgId, values) => {
					this.postMessage.sendWOPIPostMessage(FRAME_DOCUMENT, msgId, values)
				},
			})

			window.OCA?.Files?.Sidebar?.close()
		}
		this.postMessage.registerPostMessageHandler(this.postMessageHandler)

		this.load()
	},
	beforeDestroy() {
		this.postMessage.unregisterPostMessageHandler(this.postMessageHandler)
	},
	methods: {
		async load() {
			const fileid = this.fileid ?? basename(dirname(this.source))
			const version = this.fileid ? 0 : basename(this.source)

			enableScrollLock()

			// Generate WOPI token
			const { data } = await axios.post(generateUrl('/apps/richdocuments/token'), {
				fileId: fileid, shareToken: this.shareToken, version,
			})
			Config.update('urlsrc', data.urlSrc)
			Config.update('wopi_callback_url', loadState('richdocuments', 'wopi_callback_url', ''))

			// Generate form and submit to the iframe
			const action = getWopiUrl({
				fileId: fileid + '_' + loadState('richdocuments', 'instanceId', 'instanceid') + (version > 0 ? '_' + version : ''),
				title: this.filename,
				readOnly: version > 0,
				revisionHistory: !this.isPublic,
				closeButton: !Config.get('hideCloseButton'),
			})
			this.$set(this.formData, 'action', action)
			this.$set(this.formData, 'accessToken', data.token)
			this.$nextTick(() => this.$refs.form.submit())

			this.loading = LOADING_STATE.LOADING
			this.loadingTimeout = setTimeout(() => {
				console.error('Document loading failed due to timeout: Please check for failing network requests')
				this.loading = LOADING_STATE.FAILED
				this.error = t('richdocuments', 'Failed to load {productName} - please try again later', { productName: loadState('richdocuments', 'productName', 'Nextcloud Office') })
			}, (getCapabilities().richdocuments.config.timeout * 1000 || 15000))
		},
		sendPostMessage(msgId, values = {}) {
			this.postMessage.sendWOPIPostMessage(FRAME_DOCUMENT, msgId, values)
		},
		documentReady() {
			this.loading = LOADING_STATE.DOCUMENT_READY
			clearTimeout(this.loadingTimeout)
			this.sendPostMessage('Host_PostmessageReady')
			this.sendPostMessage('Insert_Button', {
				id: 'Open_Local_Editor',
				imgurl: window.location.protocol + '//' + getNextcloudUrl() + imagePath('richdocuments', 'launch.svg'),
				mobile: false,
				label: t('richdocuments', 'Open in local editor'),
				hint: t('richdocuments', 'Open in local editor'),
				insertBefore: 'print',
			})
		},
		async share() {
			FilesAppIntegration.share()
		},
		close() {
			FilesAppIntegration.close()
			if (this.modified) {
				FilesAppIntegration.updateFileInfo(undefined, Date.now())
			}
			disableScrollLock()
			this.$parent.close()
		},
		reload() {
			this.loading = LOADING_STATE.LOADING
			this.load()
			this.$refs.documentFrame.contentWindow.location.replace(this.iframeSrc)
		},
		postMessageHandler({ parsed }) {
			const { msgId, args, deprecated } = parsed
			console.debug('[viewer] Received post message', msgId, args, deprecated)
			if (deprecated) {
				return
			}

			switch (msgId) {
			case 'App_LoadingStatus':
				if (args.Status === 'Frame_Ready') {
					// defer showing the frame until collabora has finished also loading the document
					this.loading = LOADING_STATE.FRAME_READY
					this.$emit('update:loaded', true)
					FilesAppIntegration.initAfterReady()
				} else if (args.Status === 'Document_Loaded') {
					this.documentReady()
				} else if (args.Status === 'Failed') {
					this.loading = LOADING_STATE.FAILED
					this.$emit('update:loaded', true)
				}
				break
			case 'Action_Load_Resp':
				if (args.success) {
					this.documentReady()
				} else {
					this.error = args.errorMsg
					this.loading = LOADING_STATE.FAILED
				}
				break
			case 'UI_Close':
				this.close()
				break
			case 'Get_Views_Resp':
			case 'Views_List':
				this.views = args
				this.unlockAndOpenLocally()
				break
			case 'UI_SaveAs':
				this.saveAs(args.format)
				break
			case 'Action_Save_Resp':
				if (args.fileName !== this.filename) {
					FilesAppIntegration.saveAs(args.fileName)
				}
				break
			case 'UI_InsertGraphic':
				FilesAppIntegration.insertGraphic((filename, url) => {
					this.postMessage.sendWOPIPostMessage(FRAME_DOCUMENT, 'Action_InsertGraphic', {
						filename,
						url,
					})
				})
				break
			case 'UI_Mention':
				this.uiMention(parsed.args.text)
				break
			case 'UI_CreateFile':
				FilesAppIntegration.createNewFile(args.DocumentType)
				break
			case 'File_Rename':
				FilesAppIntegration.rename(args.NewName)
				break
			case 'UI_FileVersions':
				FilesAppIntegration.showRevHistory()
				break
			case 'App_VersionRestore':
				if (args.Status === 'Pre_Restore_Ack') {
					this.handlePreRestoreAck()
				}
				break
			case 'UI_Share':
				this.share()
				break
			case 'UI_ZoteroKeyMissing':
				this.showZotero = true
				break
			case 'UI_PickLink':
				this.pickLink()
				break
			case 'Action_GetLinkPreview':
				this.resolveLink(args.url)
				break
			case 'Action_Save':
				if (this.modified) {
					FilesAppIntegration.updateFileInfo(undefined, Date.now())
				}
				break
			case 'Clicked_Button':
				this.buttonClicked(args)
				break
			case 'Doc_ModifiedStatus':
				if (args.Modified !== this.modified) {
					FilesAppIntegration.updateFileInfo(undefined, Date.now())
				}
				this.modified = args.Modified
				break
			}
		},

		async buttonClicked(args) {
			if (args?.Id === 'Open_Local_Editor') {
				this.startOpenLocalProcess()
			}
		},

	},
}
</script>
<style lang="scss" scoped>
.office-viewer {
	z-index: 100000;
	max-width: 100%;
	display: flex;
	flex-direction: column;
	background-color: var(--color-main-background);

	&__loading-overlay:not(.viewer__file--hidden) {
		border-top: 3px solid var(--color-primary-element);
		display: flex;
		height: 100%;
		width: 100%;
		z-index: 1;
		top: 0;
		left: 0;
		background-color: var(--color-main-background);
		&.debug {
			opacity: .5;
		}

		::v-deep .empty-content p {
			text-align: center;
		}

		.empty-content {
			align-self: center;
			flex-grow: 1;
		}
	}

	&__iframe {
		width: 100%;
		flex-grow: 1;
	}
}
</style>

<style lang="scss">
.viewer .office-viewer:not(.viewer__file--hidden) {
	width: 100%;
	height: 100vh;
	height: 100dvh;
	top: -50px;
	position: absolute;
}
</style>
