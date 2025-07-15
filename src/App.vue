<script setup lang="ts">
import { ref } from 'vue';
// ---------------------------------------------------
// 1. 定数と型定義
// ---------------------------------------------------

// Launchpad Proのパッドのグリッドサイズ
const GRID_SIZE_X = 10;
const GRID_SIZE_Y = 10;

// Launchpad ProのLEDカラー（RGB値、0-63）
interface LPP_RGB {
  r: number;
  g: number;
  b: number;
}

// パッドの座標（0-7の範囲）
interface PadCoordinate {
  x: number;
  y: number;
}

// LEDカラーの定義 (白色と消灯のみ)
const LED_COLORS = {
  "OFF": { r: 0, g: 0, b: 0 },         // 消灯（黒）
  "WHITE": { r: 63, g: 63, b: 63 },    // 明るい白色
  "GRAY": { r: 10, g: 10, b: 10 },     // 暗い白色
  "PINK": { r: 63, g: 8, b: 12 },        // ピンク
  "DARKPINK": { r: 16, g: 2, b: 2 },   // ダークピンク
  "GREEN": { r: 0, g: 63, b: 0 },      // 緑
  "DARKGREEN": { r: 0, g: 16, b: 0 },  // ダークグリーン
  "PURPLE": { r: 63, g: 0, b: 63 },     // パープル
  "DARKPURPLE": { r: 16, g: 0, b: 16 }, // ダークパープル
  "ORANGE": { r: 63, g: 16, b: 0 },     // オレンジ
  "DARKORANGE": { r: 16, g: 4, b: 0 },  // ダークオレンジ
  "YELLOW": { r: 63, g: 63, b: 0 },     // 黄色
  "DARKYELLOW": { r: 16, g: 16, b: 0 }, // ダークイエロー
};

//各格子のスタート地点
const TARTOC_START_X: number[] = [1,1];
const TARTOC_START_Y: number[] = [1,5];

const TARTOC_ORIGIN_X: number[] = [6,6];
const TARTOC_ORIGIN_Y: number[] = [2,7];
//上下の格子のY座標差
const Y_TARTOC_OFFSET = 4;

const OCTARVE_IN_41EDO = 41;
const MIDI_CENTER_C_NOTE = ref(60);
const octaveShift = ref(0);
const transposeShift = ref(0);

// ---------------------------------------------------
// 2. MIDIノートとXY座標の相互マッピング (新規作成)
// Launchpad Pro Programmer's Reference Guideの「図4: Note レイアウト MIDI値(10進数)」に基づく
// ---------------------------------------------------

// MIDIノート番号から {x, y} 座標へのマップ
const midiNoteToXYMap = new Map<number, PadCoordinate>();
// {x, y} 座標から MIDIノート番号へのマップ (キーは "x,y" 形式の文字列)
const xyToMidiNoteMap = new Map<string, number>();

/**
 * MIDIノートとXY座標の相互マッピングを初期化する関数
 * アプリケーション起動時に一度だけ呼び出す
 */
function initializePadMappings(): void {
  // マッピングをクリア (再初期化のため)
  midiNoteToXYMap.clear();
  xyToMidiNoteMap.clear();

  // 1. メインの8x8グリッドの四角いパッド (X:1-8, Y:1-8)
  // 左下のパッド(x=1, y=1)がMIDIノート11
  for (let y = 1; y <= 8; y++) {
    for (let x = 1; x <= 8; x++) {
      const midiNote = 11 + (x - 1) + (y - 1) * 10;
      const coord: PadCoordinate = { x, y };
      midiNoteToXYMap.set(midiNote, coord);
      xyToMidiNoteMap.set(`${x},${y}`, midiNote);
    }
  }

  // 2. 周りの丸いボタン (図4参照)

  // 2.1. 下の丸ボタン列 (Y=0)
  // X:1から8
  for (let x = 1; x <= 8; x++) {
    const midiNote = x; // ノート1から8
    const coord: PadCoordinate = { x, y: 0 };
    midiNoteToXYMap.set(midiNote, coord);
    xyToMidiNoteMap.set(`${x},0`, midiNote);
  }

  // 2.2. 左の丸ボタン列 (X=0, Y:1から8)
  // Note 10(Record), 20(Double), ..., 80(Shift)
  for (let y = 1; y <= 8; y++) {
    const midiNote = y * 10; // Y*10のパターン
    const coord: PadCoordinate = { x: 0, y };
    midiNoteToXYMap.set(midiNote, coord);
    xyToMidiNoteMap.set(`${0},${y}`, midiNote);
  }

  // 2.3. 右の丸ボタン列 (X=9, Y:1から8)
  // Note 19, 29, ..., 89
  for (let y = 1; y <= 8; y++) {
    const midiNote = (y * 10) + 9; // Y*10+9のパターン
    const coord: PadCoordinate = { x: 9, y };
    midiNoteToXYMap.set(midiNote, coord);
    xyToMidiNoteMap.set(`${9},${y}`, midiNote);
  }

  // 2.4. 上の丸ボタン列 (Y=9) - 特定のノート番号を持つボタンのみ
  // (X=0, Y=9) SetupボタンはLED制御できないため除外。
  // (X=5, Y=9) Note 95 (Up Arrow)
  xyToMidiNoteMap.set(`5,9`, 95);
  midiNoteToXYMap.set(95, { x: 5, y: 9 });
  // (X=6, Y=9) Note 96 (Down Arrow)
  xyToMidiNoteMap.set(`6,9`, 96);
  midiNoteToXYMap.set(96, { x: 6, y: 9 });
  // (X=7, Y=9) Note 97 (Left Arrow)
  xyToMidiNoteMap.set(`7,9`, 97);
  midiNoteToXYMap.set(97, { x: 7, y: 9 });
  // (X=8, Y=9) Note 98 (Right Arrow)
  xyToMidiNoteMap.set(`8,9`, 98);
  midiNoteToXYMap.set(98, { x: 8, y: 9 });

  console.log("Launchpad Proのパッドマッピングが初期化されました。");
  // console.log("midiNoteToXYMap:", midiNoteToXYMap); // デバッグ用
  // console.log("xyToMidiNoteMap:", xyToMidiNoteMap); // デバッグ用
}


/**
 * 座標が丸ボタンか否かを判定する関数
 * (X,Yが0か9の場合、丸ボタンと判定)
 */
function isRoundButton(coord: PadCoordinate): boolean {
  return coord.x === 0 || coord.x === 9 || coord.y === 0 || coord.y === 9;
}

/**
 * MIDIノート番号からパッドの座標を計算
 * 今回はマッピングテーブルを使用
 */
function getPadCoordinateFromMidiNote(midiNote: number): PadCoordinate | null {
  return midiNoteToXYMap.get(midiNote) || null;
}

/**
 * パッドの座標からLEDを光らせるためのMIDIノート番号を取得する関数
 * 今回はマッピングテーブルを使用
 */
function getLedNoteFromPadCoordinate(padCoord: PadCoordinate): number | null {
  return xyToMidiNoteMap.get(`${padCoord.x},${padCoord.y}`) || null;
}


/**
 * パッドの色を計算してLaunchpad Proに送信するためのMIDIメッセージを生成
 * SysExメッセージを使用してRGBでLEDを制御する
 * @param padCoord 光らせたいパッドの座標
 * @param targetColor 設定したい色
 * @returns Launchpad Proに送信するMIDIメッセージ（Uint8Arrayなど）
 */
function generateLedColorMidiMessage(padCoord: PadCoordinate, targetColor: LPP_RGB): Uint8Array | null {
  const ledNote = getLedNoteFromPadCoordinate(padCoord);
  if (ledNote === null) {
    console.warn(`座標 (${padCoord.x}, ${padCoord.y}) に対応するLEDノートが見つかりません。`);
    return null;
  }

  // Launchpad ProのSysExメッセージ形式に合わせてUint8Arrayを生成
  // F0h 00h 20h 29h 02h 10h 0Bh (RGB) <LED Index> <Red> <Green> <Blue> F7h
  // <LED Index> は光らせたいボタンのMIDIノート番号
  const sysexHeader = [0xF0, 0x00, 0x20, 0x29, 0x02, 0x10, 0x0B]; // RGB Mode
  const sysexData = [ledNote, targetColor.r, targetColor.g, targetColor.b];
  const sysexFooter = [0xF7];

  return new Uint8Array([...sysexHeader, ...sysexData, ...sysexFooter]);
}
function setLedColor(padCoord: PadCoordinate, targetColor: LPP_RGB): void {
  const ledMessage = generateLedColorMidiMessage(padCoord, targetColor);
  if (ledMessage && launchpadOutput) {
    launchpadOutput.send(ledMessage);
    console.log(`Launchpad LEDを更新 (X:${padCoord.x}, Y:${padCoord.y}, Color:${targetColor.r},${targetColor.g},${targetColor.b})`);
  } else if (!ledMessage) {
    console.warn(`LEDメッセージ生成失敗、またはLaunchpad Outputが未設定のため、LEDを更新できませんでした。`);
  }
}

/**
 * 全てのパッドのLEDを消灯する
 * (アプリケーション起動時などに呼び出すことを想定)
 */
function turnOffAllPads(): void {
  for (let y = 0; y < GRID_SIZE_Y; y++) {
    for (let x = 0; x < GRID_SIZE_X; x++) {
      const padCoord: PadCoordinate = { x, y };
      const ledMessage = generateLedColorMidiMessage(padCoord, LED_COLORS.OFF); 
      if (ledMessage && launchpadOutput) {
        launchpadOutput.send(ledMessage);
      }
    }
  }
  console.log("全てのパッドを消灯しました。");
}

// ---------------------------------------------------
// 4. MIDIイベントハンドラ (シンプル化)
// ---------------------------------------------------

let midiAccess: MIDIAccess | null = null;
let launchpadInput: MIDIInput | null = null;
let launchpadOutput: MIDIOutput | null = null;
let virtualMidiOutput: MIDIOutput | null = null; // DAWに送る仮想MIDIデバイス

/**
 * MIDIインターフェースにアクセスし、利用可能なデバイスを列挙する関数
 */
async function initializeMidi(): Promise<void> {
  try {
    midiAccess = await navigator.requestMIDIAccess({ sysex: true });

    console.log("MIDIアクセスに成功しました！");
    
    launchpadInput = Array.from(midiAccess.inputs.values()).find(
      (input) => input.name?.includes("MIDIIN2")
    ) || null;

    launchpadOutput = Array.from(midiAccess.outputs.values()).find(
      (output) => output.name?.includes("MIDIOUT2")
    ) || null;

    virtualMidiOutput = Array.from(midiAccess.outputs.values()).find(
      (output) => output.name?.includes("Chalaxata 1") || output.name?.includes("IAC Driver")
    ) || null;


    if (launchpadInput && launchpadOutput && virtualMidiOutput) {
      console.log(`\nLaunchpad Pro 入力: ${launchpadInput.name} に接続`);
      console.log(`Launchpad Pro 出力: ${launchpadOutput.name} に接続`);
      console.log(`仮想MIDI出力: ${virtualMidiOutput.name} に接続`);

      // マッピングの初期化をここで行う
      initializePadMappings(); 

      setupMidiInputListener();
      
      // 初期化時に全てのパッドを消灯
      turnOffAllPads();
      
    } else {
      console.warn("必要なMIDIデバイスが見つかりませんでした。");
      if (!launchpadInput) console.warn("Launchpad Pro入力ポートが見つかりません。");
      if (!launchpadOutput) console.warn("Launchpad Pro出力ポートが見つかりません。");
      if (!virtualMidiOutput) console.warn("仮想MIDI出力ポートが見つかりません。");
    }

  } catch (error) {
    console.error("MIDIアクセスに失敗しました:", error);
  }
}

/**
 * MIDI入力イベントリスナーを設定する関数
 */
function setupMidiInputListener(): void {
  if (!launchpadInput) {
    console.warn("Launchpad Pro入力ポートが設定されていません。");
    return;
  }

  if (launchpadInput) {
    console.log("Launchpad Pro入力オブジェクトが存在します。リスナーを設定します。");
    launchpadInput.onmidimessage = (event: MIDIMessageEvent) => {
      const data = event.data;
      if(data==null||data.length < 3) {
        console.log("データが無いか、不正です。");
        return;
      }
      const statusByte = data[0];
      const midiNote = data[1]; // ノートオン/オフの場合のノート番号
      const velocity = data[2]; // ノートオン/オフの場合のベロシティ

      // 受け取った全てのMIDIメッセージをログに出力
      console.log(`MIDIイベント: Status: ${statusByte.toString(16)}, Data1: ${midiNote !== undefined ? midiNote.toString(16) : 'N/A'}, Data2: ${velocity !== undefined ? velocity.toString(16) : 'N/A'}`);
    
      // Note Onメッセージ (0x90) かつベロシティが0より大きい場合
      if ((statusByte & 0xF0) === 0x90 && velocity > 0) {
        handlePadPress(midiNote, velocity);
      } 
      // Note Offメッセージ (0x80) またはベロシティが0のNote On
      else if ((statusByte & 0xF0) === 0x80 || ((statusByte & 0xF0) === 0x90 && velocity === 0)) {
        handlePadRelease(midiNote);
      }
      // MIDI CC (Control Change) メッセージは、シンプルにするため引き続きコメントアウト
      // else if ((statusByte & 0xF0) === 0xB0) {
      //   // handleControlChange(statusByte, data1, data2); // CC処理が必要な場合はこの行を有効にする
      //   console.log(`MIDI CCメッセージ受信: Status=${statusByte.toString(16)}, CC=${data1}, Value=${data2}`);
      // }
    };
    console.log("Launchpad Pro入力リスナーが設定されました。");
  } else {
      console.log("エラー: Launchpad Pro入力オブジェクトが見つかりません。リスナーを設定できません。");
  }
}

/**
 * パッドが押された時のハンドラ
 * @param originalMidiNote Launchpad Proから送られてきたMIDIノート番号
 * @param velocity ベロシティ
 */
function handlePadPress(originalMidiNote: number, velocity: number): void {
  console.log(`パッド ${originalMidiNote} が押されました (ベロシティ: ${velocity})`);
  
  const padCoord = getPadCoordinateFromMidiNote(originalMidiNote); 
  if (!padCoord) {
    console.log(`MIDIノート ${originalMidiNote} からパッド座標を取得できませんでした(マッピング外)。`);
    return
  };
  console.log(`パッド座標: (${padCoord.x}, ${padCoord.y})`);

  // Launchpad ProのLEDを白色に光らせる
  const color:string = "WHITE";
  setLedColor(padCoord, LED_COLORS[color]);
  
  // TODO: calculateOutputMidiNote を呼び出して変換後のノート番号を取得
  // 仮想MIDIデバイスにMIDIノートを送信
  const outputMidiNote = convertMidiNoteToChalaxata(padCoord, 0);
  if(outputMidiNote === null) {
    console.log(`変換後のMIDIノート ${outputMidiNote} を取得できませんでした(マッピング外)。`);
    return
  }
  if (virtualMidiOutput) {
    virtualMidiOutput.send([0x90, outputMidiNote, velocity]);
    console.log(`仮想MIDIへノート ${outputMidiNote} を送信`);
  }
}

/**
 * パッドが離された時のハンドラ
 * @param originalMidiNote Launchpad Proから送られてきたMIDIノート番号
 */
function handlePadRelease(originalMidiNote: number): void {
  console.log(`パッド ${originalMidiNote} が離されました`);

  const padCoord = getPadCoordinateFromMidiNote(originalMidiNote);
  if (!padCoord) return;

  // パッドを消灯（黒色）に戻す
  // 新しい setLedColor 関数を使う
  setLedColor(padCoord, LED_COLORS.OFF);

  // TODO: calculateOutputMidiNote を呼び出して変換後のノート番号を取得
  // 仮想MIDIデバイスにMIDIノートを送信
  const outputMidiNote = convertMidiNoteToChalaxata(padCoord, 0);
  if(outputMidiNote === null) {
    console.log(`変換後のMIDIノート ${outputMidiNote} を取得できませんでした(マッピング外)。`);
    return
  }
  if (virtualMidiOutput) {
    virtualMidiOutput.send([0x80, outputMidiNote, 0]); // ノートオフはベロシティ0
    console.log(`仮想MIDIへノート ${outputMidiNote} のオフを送信`);
  }
}

// ---------------------------------------------------
// 5. アプリケーション開始
// ---------------------------------------------------

// アプリケーションが起動したらMIDI初期化関数を呼び出す
initializeMidi();

//---------------------------------------------------
// 5. tartoc(格子)の実装
// ---------------------------------------------------
const INCREMENT_OPTIONS = [0, 41, 24, 13, 33, 19, 29]; //左から順に、0次元、1次元、2次元、3次元、4次元、5次元、6次元
let dimensionIdxHorizontal = 2; //軸に対応する次元の設定
let dimensionIdxVertical = 3; // 初期設定
function getDistanceFromTartocOrigin(coord: PadCoordinate, originIdx: number): PadCoordinate {
  const origin: PadCoordinate = {x:TARTOC_ORIGIN_X[originIdx], y:TARTOC_ORIGIN_Y[originIdx]};
  console.log(`origin: (${origin.x}, ${origin.y})`);
  const distanceX = coord.x - origin.x;
  const distanceY = coord.y - origin.y;
  return {x: distanceX, y: distanceY};
}
function getMidiNoteDiffFromTartocOrigin(coord: PadCoordinate, originIdx: number): number {
  const distance = getDistanceFromTartocOrigin(coord, originIdx);
  const step = distance.x * INCREMENT_OPTIONS[dimensionIdxHorizontal] + distance.y * INCREMENT_OPTIONS[dimensionIdxVertical];
  console.log(`distance: (${distance.x}, ${distance.y}) step: ${step}`);
  return step;
}
function convertMidiNoteToChalaxata(coord: PadCoordinate, originIdx: number): number | null {
  const coordDistance: PadCoordinate = getDistanceFromTartocOrigin(coord, originIdx);
  const noteDiff = getMidiNoteDiffFromTartocOrigin(coord, originIdx);
  let outputMidiNote = noteDiff + MIDI_CENTER_C_NOTE.value;//中央ノートからの移動量なので、実際の値を出す
  outputMidiNote += transposeShift.value;//トランスポーズを適用
  outputMidiNote += octaveShift.value * OCTARVE_IN_41EDO;//オクターブシフトを適用
  console.log(outputMidiNote);
  if (outputMidiNote < 0) {
    console.log("使用可能な音程を超えています。");
    return null;
  };
  if (noteDiff > 127) {
    console.log("使用可能な音程を超えています。");
    return null;
  }
  return outputMidiNote;
}

</script>

<template>
  <button @click="initializeMidi">MIDI初期化</button>
</template>

<style scoped>
.logo {
  height: 6em;
  padding: 1.5em;
  will-change: filter;
  transition: filter 300ms;
}
.logo:hover {
  filter: drop-shadow(0 0 2em #646cffaa);
}
.logo.vue:hover {
  filter: drop-shadow(0 0 2em #42b883aa);
}
</style>
