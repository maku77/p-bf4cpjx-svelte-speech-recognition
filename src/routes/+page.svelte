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
	<span class="blue">{finalTranscript}</span>
	<span class="red">{interimTranscript}</span>
</div>
<div style="text-align: right">
	<span class="blue">青: 認識完了</span> / <span class="red">赤: 認識中</span>
</div>
{#if errorMessage}
	<div class="red">{errorMessage}</div>
{/if}

<style>
	h1 {
		font-size: 1.5rem;
	}

	.transcript-area {
		border: double 3px gray;
		height: 150px;
	}

	.blue {
		color: #66f;
	}

	.red {
		color: #f66;
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
