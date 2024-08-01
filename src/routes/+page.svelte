<script lang="ts">
	import { onMount } from 'svelte';

	let recognition: SpeechRecognition | null = null; // 音声認識 API
	let interimTranscript = ''; // 暫定の認識結果（画面表示用）
	let finalTranscript = ''; // 最終的な認識結果（画面表示用）
	let isRecording = false; // 音声認識中かどうか（ボタンの機能を「開始／停止」で切り替えるため）
	let isButtonDisabled = false; // 音声認識の初期化中はボタンを無効化する
	let errorMessage = ''; // エラーメッセージ

	onMount(() => {
		recognition = createSpeechRecognition();
		if (recognition === null) {
			console.error('SpeechRecognition API is not supported in this browser.');
			return;
		}
		recognition.onresult = handleRecognitionResult;
		recognition.onerror = handleRecognitionError;
		recognition.onstart = () => {
			isRecording = true;
			isButtonDisabled = false;
		};
		recognition.onend = () => {
			isRecording = false;
			isButtonDisabled = false;
		};
	});

	/**
	 * SpeechRecognition インスタンスを生成し基本的なパラメーター設定を行います。
	 * ブラウザーが対応していない場合は null を返します。
	 */
	function createSpeechRecognition(): SpeechRecognition | null {
		// ブラウザによって SpeechRecognition インスタンスの生成方法が異なる
		const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;

		if (!SpeechRecognition) {
			return null;
		}

		const recognition = new SpeechRecognition();
		recognition.continuous = true; // 発声が途切れてもキャプチャを終了しない
		recognition.interimResults = true; // 認識途中の結果を得る
		recognition.lang = 'ja-JP'; // 認識対象の言語
		recognition.maxAlternatives = 1; // 認識結果の候補の最大数

		return recognition;
	}

	/**
	 * 音声認識処理が進むたびに呼び出されるイベントハンドラー。
	 */
	function handleRecognitionResult(event: SpeechRecognitionEvent) {
		let interim = '';
		for (let i = event.resultIndex; i < event.results.length; i++) {
			const tr = event.results[i][0].transcript; // i 番目のチャンクの認識結果
			if (event.results[i].isFinal) {
				finalTranscript += tr; // 最終的な認識結果として表示
			} else {
				interim += tr; // 暫定の認識結果として表示
			}
		}
		interimTranscript = interim; // UI 更新
	}

	function handleRecognitionError(event: SpeechRecognitionErrorEvent) {
		console.error(`Error: ${event.error}`);
		if (event.error === 'no-speech') {
			errorMessage = '音声が検出されませんでした。もう一度お試しください。';
		} else {
			errorMessage = `エラーが発生しました: ${event.error}`;
		}
		isRecording = false;
		isButtonDisabled = false;
	}

	function startRecognition() {
		interimTranscript = '';
		finalTranscript = '';
		errorMessage = '';
		isButtonDisabled = true; // ボタンを一時的に無効化（連打防止）
		recognition?.start();
	}

	function stopRecognition() {
		isButtonDisabled = true; // ボタンを一時的に無効化（連打防止）
		recognition?.stop();
	}

	function handleButtonClick() {
		if (isRecording) {
			stopRecognition();
		} else {
			startRecognition();
		}
	}
</script>

<h1>ブラウザによる音声認識のテスト</h1>
<div style="text-align: center; margin-bottom: 1rem;">
	<button on:click={handleButtonClick} disabled={isButtonDisabled}>
		音声認識を{#if isRecording}停止{:else}開始{/if}
		{#if isRecording}
			<span class="recording-indicator"></span>
		{/if}
	</button>
</div>
<div class="transcript-area">
	{#if finalTranscript === '' && interimTranscript === ''}
		<span class="placeholder">ここに認識結果が表示されます</span>
	{/if}
	<span class="final">{finalTranscript}</span>
	<span class="interim">{interimTranscript}</span>
</div>
<div style="text-align: right">
	<span class="final">青: 認識完了</span> / <span class="interim">赤: 認識途中</span>
	<a
		style="vertical-align: middle;"
		href="https://github.com/maku77/p-bf4cpjx-svelte-speech-recognition/"
		target="_blank"
		rel="noopener"
	>
		<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 16 16" width="16" height="16"
			><path
				d="M8 0c4.42 0 8 3.58 8 8a8.013 8.013 0 0 1-5.45 7.59c-.4.08-.55-.17-.55-.38 0-.27.01-1.13.01-2.2 0-.75-.25-1.23-.54-1.48 1.78-.2 3.65-.88 3.65-3.95 0-.88-.31-1.59-.82-2.15.08-.2.36-1.02-.08-2.12 0 0-.67-.22-2.2.82-.64-.18-1.32-.27-2-.27-.68 0-1.36.09-2 .27-1.53-1.03-2.2-.82-2.2-.82-.44 1.1-.16 1.92-.08 2.12-.51.56-.82 1.28-.82 2.15 0 3.06 1.86 3.75 3.64 3.95-.23.2-.44.55-.51 1.07-.46.21-1.61.55-2.33-.66-.15-.24-.6-.83-1.23-.82-.67.01-.27.38.01.53.34.19.73.9.82 1.13.16.45.68 1.31 2.69.94 0 .67.01 1.3.01 1.49 0 .21-.15.45-.55.38A7.995 7.995 0 0 1 0 8c0-4.42 3.58-8 8-8Z"
			></path></svg
		>
	</a>
</div>
{#if errorMessage}
	<div class="error">{errorMessage}</div>
{/if}

<style>
	h1 {
		font-size: 1.5rem;
	}

	.transcript-area {
		padding: 0 0.2em;
		border: double 3px gray;
		height: 130px;
		overflow-y: auto;
	}

	.placeholder {
		color: #999;
	}

	.final {
		color: #03f;
	}

	.interim {
		background: #eee;
		color: #f36;
		text-decoration: underline;
	}

	.error {
		color: red;
	}

	.recording-indicator {
		width: 10px;
		height: 10px;
		background-color: red;
		border-radius: 50%;
		animation: blink 1s infinite;
		display: inline-block;
	}

	@keyframes blink {
		0%,
		100% {
			opacity: 1;
		}
		50% {
			opacity: 0;
		}
	}
</style>
