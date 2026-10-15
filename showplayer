/*
 * showplayer.js — shared by admin.html and index.html
 *
 * Plays a pre-programmed light show. The admin writes two small things to Firebase:
 *   classroomSync/showData  { id, json }  -> the cue list (written once when a show is loaded)
 *   classroomSync/showCtl   { state, songZero, offsetMs, showId, seq }  -> play / pause / stop
 *
 * "songZero" is the Firebase SERVER time (ms) at which the song's 0:00 plays. Every phone
 * corrects its own clock using Firebase's server-time offset and fires each cue locally
 * when (serverNow - songZero - offsetMs) reaches the cue time. No per-beat network messages,
 * so a Wi-Fi hiccup can't drop or bunch up beats.
 *
 * Cue format: [ms, kind, a, b]
 *   kind 0 = flash        a = palette index, b = duration in ms (0 = use the normal flash duration)
 *   kind 1 = glow         a = palette index, b = fade-in time in ms (screen only, stays until changed)
 *   kind 2 = dark         a = fade time in ms (screen back to idle)
 *
 * SAFETY: no more than 3 flashes in any one second, enforced here no matter what the cue list says.
 */
(function (root) {
    'use strict';

    var MIN_FLASH_GAP_MS = 340;   // hard cap: ~2.9 flashes per second maximum
    var STALE_FLASH_MS = 220;     // a flash this late is skipped rather than fired out of time

    // First index whose cue time is strictly greater than t (cues are sorted by time)
    function firstIndexAfter(cues, t) {
        var lo = 0, hi = cues.length;
        while (lo < hi) {
            var mid = (lo + hi) >> 1;
            if (cues[mid][0] <= t) lo = mid + 1; else hi = mid;
        }
        return lo;
    }

    function create(opts) {
        var db = opts.db;
        var setTorch = opts.setTorch || function () {};
        var idleBg = opts.idleBg || '#000';
        var getFlashDuration = opts.getFlashDuration || function () { return 80; };
        var onStatus = opts.onStatus || function () {};

        var serverOffset = 0;
        var ctl = null, cues = null, palette = [], title = '';
        var nextIdx = 0, enabled = false;
        var baseBg = idleBg, flashTimer = null, fadeTimer = null;

        db.ref('.info/serverTimeOffset').on('value', function (s) { serverOffset = s.val() || 0; });
        function now() { return Date.now() + serverOffset; }

        // Song position (ms) that the lights should be showing right now
        function songMs() { return now() - ctl.songZero - (ctl.offsetMs || 0); }

        function resync() {
            if (ctl && cues && ctl.state === 'playing') nextIdx = firstIndexAfter(cues, songMs());
        }

        function resetDisplay() {
            clearTimeout(flashTimer); clearTimeout(fadeTimer);
            setTorch(false);
            baseBg = idleBg;
            document.body.style.transition = '';
            document.body.style.background = idleBg;
        }

        db.ref('classroomSync/showCtl').on('value', function (snap) {
            var prev = ctl;
            ctl = snap.val();
            if (!ctl || ctl.state !== 'playing') {
                if (prev && prev.state === 'playing') resetDisplay();
            } else {
                resync();
            }
        });

        db.ref('classroomSync/showData').on('value', function (snap) {
            var v = snap.val();
            if (!v || !v.json) { cues = null; return; }
            try {
                var d = JSON.parse(v.json);
                cues = d.cues; palette = d.palette || []; title = d.title || '';
                resync();
            } catch (e) { cues = null; }
        });

        function color(i) { return palette[i] || '#ffffff'; }

        function flash(colorIdx, dur) {
            var p = performance.now();
            if (p - (root.__lsLastFlash || -1e9) < MIN_FLASH_GAP_MS) return;   // photosensitivity guard
            root.__lsLastFlash = p;
            clearTimeout(flashTimer);
            document.body.style.transition = 'background 0.03s';
            document.body.style.background = color(colorIdx);
            setTorch(true);
            flashTimer = setTimeout(function () {
                setTorch(false);
                document.body.style.transition = '';
                document.body.style.background = baseBg;
            }, dur > 0 ? dur : getFlashDuration());
        }

        function fade(bg, ms) {
            baseBg = bg;
            clearTimeout(fadeTimer);
            document.body.style.transition = 'background ' + ms + 'ms ease';
            document.body.style.background = bg;
            fadeTimer = setTimeout(function () { document.body.style.transition = ''; }, ms + 50);
        }

        function fire(c, lateMs) {
            if (c[1] === 0) { if (lateMs <= STALE_FLASH_MS) flash(c[2], c[3]); }
            else if (c[1] === 1) { fade(color(c[2]), c[3] || 300); }
            else if (c[1] === 2) { fade(idleBg, c[2] || 300); }
        }

        function tick() {
            if (!enabled || !ctl || ctl.state !== 'playing' || !cues) return;
            var t = songMs();
            while (nextIdx < cues.length && cues[nextIdx][0] <= t) {
                var c = cues[nextIdx++];
                fire(c, t - c[0]);
            }
        }
        setInterval(tick, 4);

        setInterval(function () {
            var s = { state: 'idle', title: title, loaded: !!cues };
            if (ctl && ctl.state === 'playing') {
                var t = now() - ctl.songZero;
                if (t < 0) { s.state = 'countdown'; s.startsIn = Math.ceil(-t / 1000); }
                else { s.state = 'playing'; s.posSec = t / 1000; }
            } else if (ctl && ctl.state === 'paused') {
                s.state = 'paused';
            }
            onStatus(s);
        }, 250);

        return {
            enable: function () { enabled = true; resync(); },
            disable: function () { enabled = false; resetDisplay(); },
            now: now,
            isPlaying: function () { return !!ctl && ctl.state === 'playing'; }
        };
    }

    var api = { create: create, firstIndexAfter: firstIndexAfter };
    if (typeof module !== 'undefined' && module.exports) module.exports = api;
    root.LightShow = api;
})(typeof window !== 'undefined' ? window : globalThis);
