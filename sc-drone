(
// ============================================================
// DRONE SYNTH V4.1
// manual-tab UI version (no TabbedView dependency)
// ============================================================

s.waitForBoot({

	// --------------------------------------------------------
	// cleanup from previous runs
	// --------------------------------------------------------
	if(~droneWin.notNil) {
		~droneWin.close;
	};

	if(~droneMidiOn.notNil) { ~droneMidiOn.free };
	if(~droneMidiOff.notNil) { ~droneMidiOff.free };

	if(~droneRecorderSynth.notNil) {
		~droneRecorderSynth.free;
		~droneRecorderSynth = nil;
	};

	if(~droneRecBuf.notNil) {
		~droneRecBuf.close;
		~droneRecBuf.free;
		~droneRecBuf = nil;
	};

	if(~droneSynth.notNil) {
		~droneSynth.set(\gate, 0);
		~droneSynth = nil;
	};

	// --------------------------------------------------------
	// MIDI setup
	// --------------------------------------------------------
	MIDIClient.init;
	MIDIIn.connectAll;

	// --------------------------------------------------------
	// parameter state
	// --------------------------------------------------------
	~droneParams = IdentityDictionary[
		\freq -> 55.0,
		\amp -> 0.14,

		\detune -> 0.18,
		\subAmp -> 0.30,
		\lowLayerAmp -> 0.28,
		\highLayerAmp -> 0.22,
		\harmAmp -> 0.36,
		\noiseAmp -> 0.02,

		\cutoff -> 1800,
		\rq -> 0.22,
		\filterLfoRate -> 0.035,
		\filterLfoDepth -> 900,

		\pitchDriftRate -> 0.05,
		\pitchDriftDepth -> 0.08,

		\saturation -> 1.5,
		\stereoWidth -> 0.65,

		\delayMix -> 0.14,
		\delayTime -> 0.28,
		\reverbMix -> 0.34,
		\room -> 0.86,
		\damp -> 0.28,

		\chordMix -> 0.48,
		\interval1 -> 7,
		\interval2 -> 12,
		\interval3 -> 19,

		\waveBlend -> 0.40,
		\motion -> 0.35,
		\morph -> 0.25,
		\shimmer -> 0.18,

		\attack -> 6.0,
		\release -> 10.0,

		\gate -> 1
	];

	~droneSpecs = IdentityDictionary[
		\freq -> [20, 220, 0.1],
		\amp -> [0.0, 0.4, 0.001],

		\detune -> [0.0, 2.0, 0.001],
		\subAmp -> [0.0, 1.0, 0.001],
		\lowLayerAmp -> [0.0, 1.0, 0.001],
		\highLayerAmp -> [0.0, 1.0, 0.001],
		\harmAmp -> [0.0, 1.0, 0.001],
		\noiseAmp -> [0.0, 0.2, 0.001],

		\cutoff -> [60, 12000, 1],
		\rq -> [0.05, 0.95, 0.001],
		\filterLfoRate -> [0.001, 1.0, 0.001],
		\filterLfoDepth -> [0, 5000, 1],

		\pitchDriftRate -> [0.001, 1.0, 0.001],
		\pitchDriftDepth -> [0.0, 0.5, 0.001],

		\saturation -> [0.5, 6.0, 0.01],
		\stereoWidth -> [0.0, 1.0, 0.001],

		\delayMix -> [0.0, 1.0, 0.001],
		\delayTime -> [0.01, 0.8, 0.001],
		\reverbMix -> [0.0, 1.0, 0.001],
		\room -> [0.0, 1.0, 0.001],
		\damp -> [0.0, 1.0, 0.001],

		\chordMix -> [0.0, 1.0, 0.001],
		\interval1 -> [-24, 24, 1],
		\interval2 -> [-24, 24, 1],
		\interval3 -> [-24, 24, 1],

		\waveBlend -> [0.0, 1.0, 0.001],
		\motion -> [0.0, 1.0, 0.001],
		\morph -> [0.0, 1.0, 0.001],
		\shimmer -> [0.0, 1.0, 0.001],

		\attack -> [0.01, 20.0, 0.01],
		\release -> [0.01, 20.0, 0.01]
	];

	// --------------------------------------------------------
	// recorder synthdef
	// --------------------------------------------------------
	SynthDef(\droneRecorderTap, { |bus = 0, bufnum = 0|
		var sig;
		sig = In.ar(bus, 2);
		DiskOut.ar(bufnum, sig);
	}).add;

	// --------------------------------------------------------
	// main synthdef
	// --------------------------------------------------------
	SynthDef(\droneMachineV41, {
		|out=0, freq=55, amp=0.14,
		detune=0.18, subAmp=0.30, lowLayerAmp=0.28, highLayerAmp=0.22, harmAmp=0.36, noiseAmp=0.02,
		cutoff=1800, rq=0.22, filterLfoRate=0.035, filterLfoDepth=900,
		pitchDriftRate=0.05, pitchDriftDepth=0.08,
		saturation=1.5, stereoWidth=0.65,
		delayMix=0.14, delayTime=0.28, reverbMix=0.34, room=0.86, damp=0.28,
		chordMix=0.48, interval1=7, interval2=12, interval3=19,
		waveBlend=0.40, motion=0.35, morph=0.25, shimmer=0.18,
		attack=6, release=10, gate=1|

		var env;
		var motionRate, motionDepth;
		var drift1, drift2, drift3, drift4, drift5, drift6;
		var baseA, baseB, baseC;
		var lowLayer, midSoft, midBright, highLayer;
		var harm1, harm2, harm3;
		var sub, noise, sig;
		var filt, brightLift;
		var dry, delayed, verb, shimmerDelay;
		var left, right;

		env = EnvGen.kr(
			Env.asr(attack, 1, release),
			gate,
			doneAction: 2
		);

		motionRate = motion.linexp(0, 1, 0.003, 0.18);
		motionDepth = motion.linlin(0, 1, 0.2, 1.0);

		drift1 = LFNoise1.kr((pitchDriftRate * 0.80) + (motionRate * 0.20)).range(pitchDriftDepth.neg, pitchDriftDepth).midiratio;
		drift2 = LFNoise1.kr((pitchDriftRate * 1.00) + (motionRate * 0.10)).range(pitchDriftDepth.neg, pitchDriftDepth).midiratio;
		drift3 = LFNoise1.kr((pitchDriftRate * 1.20) + (motionRate * 0.15)).range(pitchDriftDepth.neg, pitchDriftDepth).midiratio;
		drift4 = LFNoise1.kr((pitchDriftRate * 0.75) + (motionRate * 0.08)).range(pitchDriftDepth.neg, pitchDriftDepth).midiratio;
		drift5 = LFNoise1.kr((pitchDriftRate * 1.07) + (motionRate * 0.11)).range(pitchDriftDepth.neg, pitchDriftDepth).midiratio;
		drift6 = LFNoise1.kr((pitchDriftRate * 1.28) + (motionRate * 0.13)).range(pitchDriftDepth.neg, pitchDriftDepth).midiratio;

		baseA = freq * ((detune.neg).midiratio) * drift1;
		baseB = freq * drift2;
		baseC = freq * (detune.midiratio) * drift3;

		lowLayer =
			(SinOsc.ar(baseA * 0.5) * 0.20) +
			(SinOsc.ar(baseB * 0.5) * 0.22) +
			(VarSaw.ar(baseC * 0.5, 0, 0.55) * 0.10);

		midSoft =
			(SinOsc.ar(baseA) * 0.20) +
			(SinOsc.ar(baseB) * 0.22) +
			(SinOsc.ar(baseC) * 0.20) +
			(VarSaw.ar(baseB, 0, 0.45) * 0.14) +
			(Pulse.ar(baseA, 0.48) * 0.06);

		midBright =
			(Saw.ar(baseA) * 0.15) +
			(Saw.ar(baseB) * 0.17) +
			(Saw.ar(baseC) * 0.15) +
			(VarSaw.ar(baseC, 0, 0.52) * 0.14) +
			(Pulse.ar(baseB, 0.50) * 0.07);

		highLayer =
			(SinOsc.ar(baseA * 2) * 0.05) +
			(VarSaw.ar(baseB * 2, 0, 0.40) * 0.07) +
			(Saw.ar(baseC * 2) * 0.05);

		harm1 = freq * interval1.midiratio * drift4;
		harm2 = freq * interval2.midiratio * drift5;
		harm3 = freq * interval3.midiratio * drift6;

		sub = SinOsc.ar(freq * 0.5) * subAmp;
		noise = PinkNoise.ar(noiseAmp) + (BrownNoise.ar(noiseAmp * 0.4));

		sig =
			(lowLayer * lowLayerAmp) +
			(XFade2.ar(midSoft, midBright, (waveBlend * 2) - 1)) +
			(highLayer * highLayerAmp) +
			(
				(SinOsc.ar(harm1) * 0.12) +
				(VarSaw.ar(harm2, 0, 0.50) * 0.10) +
				(Pulse.ar(harm3, 0.48) * 0.08)
			) * chordMix * harmAmp +
			sub +
			noise;

		filt = SinOsc.kr(filterLfoRate + (motionRate * 0.15)).range(
			(cutoff - (filterLfoDepth * motionDepth)).max(40),
			(cutoff + (filterLfoDepth * motionDepth)).min(18000)
		);

		sig = RLPF.ar(sig, filt, rq.clip(0.05, 0.95));

		brightLift = BPF.ar(sig, (cutoff * 1.8).clip(120, 12000), 0.25) * shimmer;
		sig = sig + brightLift;

		sig = XFade2.ar(sig, (sig * 3).tanh, (morph * 2) - 1);
		sig = (sig * saturation).tanh;

		left = DelayC.ar(sig, 0.03, stereoWidth.linlin(0, 1, 0.000, 0.015));
		right = DelayC.ar(sig, 0.03, stereoWidth.linlin(0, 1, 0.001, 0.022));

		dry = [left, right];

		delayed = CombC.ar(
			dry,
			1.0,
			delayTime.clip(0.01, 0.8),
			3 + (motion * 4)
		);

		shimmerDelay = PitchShift.ar(delayed, 0.2, 2.0, 0.01, 0.01) * (shimmer * 0.18);

		dry = XFade2.ar(dry, delayed + shimmerDelay, (delayMix * 2) - 1);

		verb = FreeVerb2.ar(
			dry[0],
			dry[1],
			reverbMix.clip(0, 1),
			room.clip(0, 1),
			damp.clip(0, 1)
		);

		Out.ar(out, LeakDC.ar(verb * amp * env));
	}).add;

	s.sync;

	~droneControlViews = IdentityDictionary.new;
	~droneUpdatingGUI = false;

	// --------------------------------------------------------
	// synth helpers
	// --------------------------------------------------------
	~droneStart = {
		if(~droneSynth.notNil) {
			~droneSynth.set(\gate, 0);
			~droneSynth = nil;
		};
		~droneSynth = Synth(\droneMachineV41, ~droneParams.asKeyValuePairs);
		"Drone started.".postln;
	};

	~droneStop = {
		if(~droneSynth.notNil) {
			~droneSynth.set(\gate, 0);
			~droneSynth = nil;
			"Drone stopped.".postln;
		};
	};

	~droneSet = { |key, value|
		if(~droneParams.includesKey(key)) {
			~droneParams[key] = value;
			if(~droneSynth.notNil) {
				~droneSynth.set(key, value);
			};
			if(~droneUpdateGUI.notNil) {
				~droneUpdateGUI.();
			};
		} {
			("Unknown param: " ++ key).warn;
		};
	};

	~droneApply = { |dict|
		dict.keysValuesDo { |k, v|
			if(~droneParams.includesKey(k)) {
				~droneParams[k] = v;
				if(~droneSynth.notNil) {
					~droneSynth.set(k, v);
				};
			};
		};
		if(~droneUpdateGUI.notNil) { ~droneUpdateGUI.() };
	};

	~dronePreset = { |name|
		switch(name,
			\dark, {
				~droneApply.((
					freq: 42, amp: 0.16, detune: 0.10, subAmp: 0.42,
					lowLayerAmp: 0.40, highLayerAmp: 0.10, harmAmp: 0.20, noiseAmp: 0.01,
					cutoff: 650, rq: 0.22, filterLfoRate: 0.02, filterLfoDepth: 350,
					pitchDriftRate: 0.03, pitchDriftDepth: 0.04,
					saturation: 1.3, stereoWidth: 0.50,
					delayMix: 0.10, delayTime: 0.36, reverbMix: 0.42, room: 0.94, damp: 0.24,
					chordMix: 0.22, interval1: 7, interval2: 12, interval3: 19,
					waveBlend: 0.15, motion: 0.18, morph: 0.12, shimmer: 0.05
				));
			},
			\shimmer, {
				~droneApply.((
					freq: 82, amp: 0.12, detune: 0.22, subAmp: 0.12,
					lowLayerAmp: 0.12, highLayerAmp: 0.30, harmAmp: 0.52, noiseAmp: 0.01,
					cutoff: 4200, rq: 0.14, filterLfoRate: 0.05, filterLfoDepth: 1600,
					pitchDriftRate: 0.07, pitchDriftDepth: 0.09,
					saturation: 1.1, stereoWidth: 0.90,
					delayMix: 0.24, delayTime: 0.21, reverbMix: 0.52, room: 0.98, damp: 0.15,
					chordMix: 0.78, interval1: 7, interval2: 12, interval3: 16,
					waveBlend: 0.62, motion: 0.58, morph: 0.10, shimmer: 0.58
				));
			},
			\industrial, {
				~droneApply.((
					freq: 50, amp: 0.13, detune: 0.70, subAmp: 0.18,
					lowLayerAmp: 0.22, highLayerAmp: 0.38, harmAmp: 0.12, noiseAmp: 0.08,
					cutoff: 1200, rq: 0.30, filterLfoRate: 0.08, filterLfoDepth: 1100,
					pitchDriftRate: 0.12, pitchDriftDepth: 0.14,
					saturation: 3.5, stereoWidth: 0.72,
					delayMix: 0.12, delayTime: 0.10, reverbMix: 0.18, room: 0.68, damp: 0.44,
					chordMix: 0.16, interval1: 1, interval2: 7, interval3: 13,
					waveBlend: 0.88, motion: 0.72, morph: 0.70, shimmer: 0.08
				));
			},
			\choir, {
				~droneApply.((
					freq: 64, amp: 0.14, detune: 0.08, subAmp: 0.18,
					lowLayerAmp: 0.18, highLayerAmp: 0.16, harmAmp: 0.58, noiseAmp: 0.008,
					cutoff: 2200, rq: 0.15, filterLfoRate: 0.024, filterLfoDepth: 500,
					pitchDriftRate: 0.03, pitchDriftDepth: 0.04,
					saturation: 1.05, stereoWidth: 0.82,
					delayMix: 0.07, delayTime: 0.28, reverbMix: 0.52, room: 0.99, damp: 0.12,
					chordMix: 0.82, interval1: 4, interval2: 7, interval3: 12,
					waveBlend: 0.24, motion: 0.20, morph: 0.06, shimmer: 0.16
				));
			}
		);
	};

	~droneRandomize = {
		~droneApply.((
			freq: rrand(30.0, 110.0),
			amp: rrand(0.10, 0.18),

			detune: rrand(0.02, 0.80),
			subAmp: rrand(0.05, 0.45),
			lowLayerAmp: rrand(0.05, 0.45),
			highLayerAmp: rrand(0.05, 0.40),
			harmAmp: rrand(0.10, 0.70),
			noiseAmp: rrand(0.0, 0.06),

			cutoff: rrand(300, 5000),
			rq: rrand(0.10, 0.35),
			filterLfoRate: rrand(0.01, 0.12),
			filterLfoDepth: rrand(100, 1800),

			pitchDriftRate: rrand(0.01, 0.15),
			pitchDriftDepth: rrand(0.01, 0.12),

			saturation: rrand(1.0, 4.0),
			stereoWidth: rrand(0.3, 0.95),

			delayMix: rrand(0.0, 0.28),
			delayTime: rrand(0.08, 0.40),
			reverbMix: rrand(0.10, 0.58),
			room: rrand(0.60, 1.0),
			damp: rrand(0.08, 0.50),

			chordMix: rrand(0.1, 0.85),
			interval1: [0, 3, 4, 5, 7, 10, 12].choose,
			interval2: [7, 10, 12, 14, 15, 16, 19].choose,
			interval3: [12, 14, 16, 17, 19, 21, 24].choose,

			waveBlend: rrand(0.0, 1.0),
			motion: rrand(0.1, 0.9),
			morph: rrand(0.0, 0.8),
			shimmer: rrand(0.0, 0.6)
		));
	};

	~droneStartRecording = {
		var path;
		if(~droneRecBuf.notNil) {
			"Already recording.".warn;
		} {
			path = Platform.userHomeDir +/+ ("drone_recording_" ++ Date.localtime.stamp ++ ".aiff");
			~droneRecBuf = Buffer.alloc(s, 65536, 2);
			~droneRecBuf.write(path, "aiff", "int16", 0, 0, true);
			s.sync;
			~droneRecorderSynth = Synth.tail(nil, \droneRecorderTap, [\bus, 0, \bufnum, ~droneRecBuf.bufnum]);
			("Recording to: " ++ path).postln;
		};
	};

	~droneStopRecording = {
		if(~droneRecorderSynth.notNil) {
			~droneRecorderSynth.free;
			~droneRecorderSynth = nil;
		};
		if(~droneRecBuf.notNil) {
			~droneRecBuf.close;
			~droneRecBuf.free;
			~droneRecBuf = nil;
			"Recording stopped.".postln;
		};
	};

	~droneMidiOn = MIDIFunc.noteOn({ |vel, num|
		var hz, a;
		hz = num.midicps;
		a = vel.linlin(1, 127, 0.08, 0.20);
		if(~droneSynth.isNil) { ~droneStart.() };
		~droneSet.(\freq, hz);
		~droneSet.(\amp, a);
	});

	~droneMidiOff = MIDIFunc.noteOff({ |vel, num| });

	// --------------------------------------------------------
	// ui helpers
	// --------------------------------------------------------
	~makeParamRow = { |parent, y, label, key|
		var spec, min, max, step, row, text, slider, box;

		spec = ~droneSpecs[key];
		min = spec[0];
		max = spec[1];
		step = spec[2];

		row = CompositeView(parent, Rect(10, y, parent.bounds.width - 20, 24));
		text = StaticText(row, Rect(0, 2, 120, 20)).string_(label);
		slider = Slider(row, Rect(125, 2, row.bounds.width - 215, 20));
		box = NumberBox(row, Rect(row.bounds.width - 85, 2, 85, 20))
			.step_(step)
			.clipLo_(min)
			.clipHi_(max);

		~droneControlViews[key] = (
			row: row,
			text: text,
			slider: slider,
			box: box,
			min: min,
			max: max
		);

		slider.action = {
			var newVal;
			if(~droneUpdatingGUI.not) {
				newVal = slider.value.linlin(0, 1, min, max);
				box.value = newVal;
				~droneSet.(key, newVal);
			};
		};

		box.action = {
			var newVal;
			if(~droneUpdatingGUI.not) {
				newVal = box.value.clip(min, max);
				box.value = newVal;
				slider.value = newVal.linlin(min, max, 0, 1).clip(0, 1);
				~droneSet.(key, newVal);
			};
		};

		row;
	};

	~layoutParamTab = { |tabView, keysAndLabels|
		var y = 10;
		keysAndLabels.do { |pair|
			~makeParamRow.(tabView, y, pair[0], pair[1]);
			y = y + 28;
		};
	};

	~updateParamRowsForParent = { |parent|
		~droneControlViews.keysValuesDo { |k, item|
			if(item[\row].parent == parent) {
				var w = parent.bounds.width - 20;
				item[\row].bounds = Rect(10, item[\row].bounds.top, w, 24);
				item[\text].bounds = Rect(0, 2, 120, 20);
				item[\box].bounds = Rect(w - 85, 2, 85, 20);
				item[\slider].bounds = Rect(125, 2, (w - 215).max(60), 20);
			};
		};
	};

	~droneUpdateGUI = {
		~droneUpdatingGUI = true;

		~droneControlViews.keysValuesDo { |k, item|
			var val = ~droneParams[k];
			item[\box].value = val;
			item[\slider].value = val.linlin(item[\min], item[\max], 0, 1).clip(0, 1);
		};

		if(~xyPad.notNil) {
			~xyPad.x_(~droneParams[\motion].clip(0, 1));
			~xyPad.y_(~droneParams[\shimmer].clip(0, 1));
		};

		~droneUpdatingGUI = false;
	};

	~showDroneTab = { |index|
		~currentDroneTab = index;
		~tabPages.do { |view, i|
			view.visible = (i == index);
		};
		~tabButtons.do { |bt, i|
			bt.value = if(i == index) { 1 } { 0 };
		};
	};

	~droneRelayoutTabs = {
		var bounds, tabBarY, pageY, pageH, pageW, buttonGap, tabCount, tabW;

		bounds = ~droneWin.view.bounds;
		~titleText.bounds = Rect(10, 8, bounds.width - 20, 24);

		tabBarY = 40;
		pageY = 74;
		pageW = bounds.width - 20;
		pageH = bounds.height - pageY - 10;

		~tabBar.bounds = Rect(10, tabBarY, bounds.width - 20, 28);

		buttonGap = 4;
		tabCount = ~tabButtons.size;
		tabW = (((~tabBar.bounds.width) - ((tabCount - 1) * buttonGap)) / tabCount).floor.max(70);

		~tabButtons.do { |bt, i|
			bt.bounds = Rect(i * (tabW + buttonGap), 0, tabW, 24);
		};

		~tabPages.do { |view|
			view.bounds = Rect(10, pageY, pageW, pageH);
		};

		~updateParamRowsForParent.(~pitchTab);
		~updateParamRowsForParent.(~filterTab);
		~updateParamRowsForParent.(~spaceTab);
		~updateParamRowsForParent.(~chordTab);

		if(~xyPad.notNil) {
			var padSize = ((~xyTab.bounds.width - 40).min(~xyTab.bounds.height - 70)).max(120);
			~xyPad.bounds = Rect(
				((~xyTab.bounds.width - padSize) / 2).max(10),
				20,
				padSize,
				padSize
			);
			~xyLabel.bounds = Rect(10, ~xyPad.bounds.bottom + 8, ~xyTab.bounds.width - 20, 20);
		};

		~transportButtons.do { |bt, i|
			var w = (((~transportTab.bounds.width - 30) / 4) - 6).max(70);
			bt.bounds = Rect(10 + (i * (w + 6)), 20, w, 24);
		};

		~presetButtons.do { |bt, i|
			var w = (((~transportTab.bounds.width - 30) / 4) - 6).max(70);
			bt.bounds = Rect(10 + (i * (w + 6)), 90, w, 24);
		};

		~recordButtons.do { |bt, i|
			var w = (((~transportTab.bounds.width - 26) / 2) - 6).max(100);
			bt.bounds = Rect(10 + (i * (w + 6)), 160, w, 24);
		};
	};

	// --------------------------------------------------------
	// window
	// --------------------------------------------------------
	~droneWin = Window("Drone Synth V4.1", Rect(80, 80, 820, 560));
	~droneWin.alwaysOnTop_(true);
	~droneWin.view.resize_(5);

	~titleText = StaticText(~droneWin, Rect(10, 8, 700, 24))
		.string_("Drone Synth V4.1")
		.font_(Font("Helvetica", 20))
		.align_(\center);

	~tabBar = CompositeView(~droneWin, Rect(10, 40, 800, 28));

	~transportTab = CompositeView(~droneWin, Rect(10, 74, 800, 470));
	~xyTab = CompositeView(~droneWin, Rect(10, 74, 800, 470));
	~pitchTab = CompositeView(~droneWin, Rect(10, 74, 800, 470));
	~filterTab = CompositeView(~droneWin, Rect(10, 74, 800, 470));
	~spaceTab = CompositeView(~droneWin, Rect(10, 74, 800, 470));
	~chordTab = CompositeView(~droneWin, Rect(10, 74, 800, 470));

	~tabPages = [~transportTab, ~xyTab, ~pitchTab, ~filterTab, ~spaceTab, ~chordTab];

	~tabButtons = [
		Button(~tabBar, Rect(0, 0, 120, 24)).states_([["Transport"], ["Transport"]]).action_({ ~showDroneTab.(0) }),
		Button(~tabBar, Rect(0, 0, 120, 24)).states_([["XY Pad"], ["XY Pad"]]).action_({ ~showDroneTab.(1) }),
		Button(~tabBar, Rect(0, 0, 120, 24)).states_([["Pitch / Mix"], ["Pitch / Mix"]]).action_({ ~showDroneTab.(2) }),
		Button(~tabBar, Rect(0, 0, 120, 24)).states_([["Filter / Motion"], ["Filter / Motion"]]).action_({ ~showDroneTab.(3) }),
		Button(~tabBar, Rect(0, 0, 120, 24)).states_([["Space / Tone"], ["Space / Tone"]]).action_({ ~showDroneTab.(4) }),
		Button(~tabBar, Rect(0, 0, 120, 24)).states_([["Chord / Env"], ["Chord / Env"]]).action_({ ~showDroneTab.(5) })
	];

	// --------------------------------------------------------
	// transport tab
	// --------------------------------------------------------
	StaticText(~transportTab, Rect(10, 0, 300, 18))
		.string_("Transport")
		.font_(Font("Helvetica", 14));

	~transportButtons = [
		Button(~transportTab, Rect(10, 20, 120, 24)).states_([["Start"]]).action_({ ~droneStart.() }),
		Button(~transportTab, Rect(136, 20, 120, 24)).states_([["Stop"]]).action_({ ~droneStop.() }),
		Button(~transportTab, Rect(262, 20, 120, 24)).states_([["Random"]]).action_({ ~droneRandomize.() }),
		Button(~transportTab, Rect(388, 20, 120, 24)).states_([["Restart"]]).action_({
			~droneStop.();
			AppClock.sched(0.15, { ~droneStart.(); nil });
		})
	];

	StaticText(~transportTab, Rect(10, 70, 300, 18))
		.string_("Presets")
		.font_(Font("Helvetica", 14));

	~presetButtons = [
		Button(~transportTab, Rect(10, 90, 120, 24)).states_([["Dark"]]).action_({ ~dronePreset.(\dark) }),
		Button(~transportTab, Rect(136, 90, 120, 24)).states_([["Shimmer"]]).action_({ ~dronePreset.(\shimmer) }),
		Button(~transportTab, Rect(262, 90, 120, 24)).states_([["Industrial"]]).action_({ ~dronePreset.(\industrial) }),
		Button(~transportTab, Rect(388, 90, 120, 24)).states_([["Choir"]]).action_({ ~dronePreset.(\choir) })
	];

	StaticText(~transportTab, Rect(10, 140, 300, 18))
		.string_("Recording")
		.font_(Font("Helvetica", 14));

	~recordButtons = [
		Button(~transportTab, Rect(10, 160, 180, 24)).states_([["Start Recording"]]).action_({ ~droneStartRecording.() }),
		Button(~transportTab, Rect(196, 160, 180, 24)).states_([["Stop Recording"]]).action_({ ~droneStopRecording.() })
	];

	StaticText(~transportTab, Rect(10, 220, 680, 20))
		.string_("MIDI note-on retunes the drone. Note-off intentionally does not stop it.")
		.align_(\left);

	// --------------------------------------------------------
	// xy tab
	// --------------------------------------------------------
	~xyPad = Slider2D(~xyTab, Rect(100, 20, 240, 240));
	~xyPad.x_(~droneParams[\motion]);
	~xyPad.y_(~droneParams[\shimmer]);

	~xyLabel = StaticText(~xyTab, Rect(10, 270, 600, 20))
		.string_("X = motion, Y = shimmer")
		.align_(\center);

	~xyPad.action = {
		~droneSet.(\motion, ~xyPad.x.clip(0, 1));
		~droneSet.(\shimmer, ~xyPad.y.clip(0, 1));
	};

	// --------------------------------------------------------
	// param tabs
	// --------------------------------------------------------
	~layoutParamTab.(~pitchTab, [
		["Base Freq", \freq],
		["Amplitude", \amp],
		["Detune", \detune],
		["Sub Level", \subAmp],
		["Low Layer", \lowLayerAmp],
		["High Layer", \highLayerAmp],
		["Harmonic Level", \harmAmp],
		["Noise Level", \noiseAmp]
	]);

	~layoutParamTab.(~filterTab, [
		["Cutoff", \cutoff],
		["RQ", \rq],
		["Filter LFO Rate", \filterLfoRate],
		["Filter LFO Depth", \filterLfoDepth],
		["Pitch Drift Rate", \pitchDriftRate],
		["Pitch Drift Depth", \pitchDriftDepth],
		["Wave Blend", \waveBlend],
		["Motion", \motion],
		["Morph", \morph],
		["Shimmer", \shimmer]
	]);

	~layoutParamTab.(~spaceTab, [
		["Saturation", \saturation],
		["Stereo Width", \stereoWidth],
		["Delay Mix", \delayMix],
		["Delay Time", \delayTime],
		["Reverb Mix", \reverbMix],
		["Room", \room],
		["Damp", \damp]
	]);

	~layoutParamTab.(~chordTab, [
		["Chord Mix", \chordMix],
		["Interval 1", \interval1],
		["Interval 2", \interval2],
		["Interval 3", \interval3],
		["Attack", \attack],
		["Release", \release]
	]);

	~droneUpdateGUI.();
	~droneRelayoutTabs.();
	~showDroneTab.(0);

	~droneWin.view.onResize_({
		~droneRelayoutTabs.();
	});

	~droneWin.onClose_({
		~droneStopRecording.();
		if(~droneMidiOn.notNil) { ~droneMidiOn.free };
		if(~droneMidiOff.notNil) { ~droneMidiOff.free };
		if(~droneSynth.notNil) {
			~droneSynth.set(\gate, 0);
			~droneSynth = nil;
		};
	});

	~droneWin.front;
	~droneStart.();

	"Drone Synth V4.1 ready.".postln;
});
)
