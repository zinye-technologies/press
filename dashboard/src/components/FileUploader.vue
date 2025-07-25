<template>
	<div>
		<input
			ref="input"
			type="file"
			:accept="fileTypes"
			class="hidden"
			@change="onFileAdd"
		/>
		<slot
			v-bind="{
				file,
				uploading,
				progress,
				uploaded,
				message,
				error,
				total,
				success,
				openFileSelector,
			}"
		/>
	</div>
</template>

<script>
import FileUploader from '@/controllers/fileUploader';
import S3FileUploader from '@/controllers/s3FileUploader';
import { trypromise } from '@/utils';

export default {
	name: 'FileUploader',
	props: [
		'fileTypes',
		'uploadArgs',
		's3',
		'type',
		'fileValidator',
		'disableAutoUpload',
	],
	emits: ['success', 'failure', 'setFile'],
	data() {
		return {
			uploader: null,
			uploading: false,
			uploaded: 0,
			error: null,
			message: '',
			total: 0,
			file: null,
			finishedUploading: false,
		};
	},
	computed: {
		progress() {
			let value = Math.floor((this.uploaded / this.total) * 100);
			return isNaN(value) ? 0 : value;
		},
		success() {
			return this.finishedUploading && !this.error;
		},
	},
	methods: {
		openFileSelector() {
			this.$refs['input'].click();
		},
		async onFileAdd(e) {
			this.error = null;
			this.finishedUploading = false;
			this.uploading = false;
			this.uploaded = 0;
			this.total = 0;
			this.message = '';

			this.file = e.target.files[0];

			if (this.file && this.fileValidator) {
				let [error, _] = await trypromise(this.fileValidator(this.file));
				if (error) {
					this.error = error;
				}
			}

			if (this.error) {
				this.$emit('failure', this.error);
				return;
			}

			this.$emit('setFile', this.file);
			if (!this.disableAutoUpload) await this.uploadFile();
		},
		async uploadFile() {
			if (this.uploaded || this.finishedUploading || this.error || !this.file) {
				return;
			}
			this.error = null;
			this.uploaded = 0;
			this.total = 0;

			this.uploader = this.s3 ? new S3FileUploader() : new FileUploader();
			this.uploader.on('start', () => {
				this.uploading = true;
			});
			this.uploader.on('progress', (data) => {
				this.uploaded = data.uploaded;
				this.total = data.total;
			});
			this.uploader.on('error', () => {
				this.uploading = false;
				this.error = 'Error Uploading File';
			});
			this.uploader.on('finish', () => {
				this.uploading = false;
				this.finishedUploading = true;
			});
			this.uploader
				.upload(this.file, this.uploadArgs || {})
				.then((data) => {
					this.$emit('success', data);
				})
				.catch((error) => {
					this.uploading = false;
					let errorMessage = 'Error Uploading File';
					if (error._server_messages) {
						errorMessage = JSON.parse(
							JSON.parse(error._server_messages)[0],
						).message;
					} else if (error.exc) {
						errorMessage = JSON.parse(error.exc)[0]
							.split('\n')
							.slice(-2, -1)[0];
					}
					this.error = errorMessage;
					this.$emit('failure', errorMessage);
				});
		},
	},
};
</script>
