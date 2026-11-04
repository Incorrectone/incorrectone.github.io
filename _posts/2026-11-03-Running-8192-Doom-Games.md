---
title: Running 8192 Doom Instances on a big big server
date: 2026-11-01 13:30:25 +/-1000
categories: [cloud, gaming]
tags: [cloud, benchmarking, aws, doom, absurd]     # TAG names should always be lowercase
author: incor

# [true/false]

toc: true 
comments: false
math: false

pin: false

image:
  path: doom-bucket/doom8192.png
  alt: Doom Mosaic with 8192 instances

---

Running DOOM (1993) on different electronic devices has been a thing on the internet for a long time. But the opposite has been lacking severely. So in this blog we try to run DOOM at scale on an AWS EC2 `c7a.32xlarge` spot instance.

You can see the final 8192 mosaic on YouTube in 4K.

{% include embed/youtube.html id='Oq6W3fukuM4' %}

## the Initial Plan and Problems

I had originally thought I could just run however many instances in parallel on a virtual screen and record the outputs. However, the audio bugged out past 32 instances and scaling the virtual screen was not viable past 128 instances, which would have rendered the project impossible.

Starting with the open-source DOOM engine, [chocolate-doom](https://github.com/chocolate-doom/chocolate-doom). On a `c7a.2xlarge` machine for development work with 40GB gp3 root volume.

```bash
$ SDL_VIDEODRIVER=dummy chocolate-doom -iwad /usr/share/games/doom/freedoom1.wad -timedemo demo1 -nosound -nomusic
```
```text
timed 7117 gametics in 219 realtics (1137.420044 fps)
```

Using Xvfb (virtual display), PulseAudio (Audio Sink), and ffmpeg to record. Also I used an S3 Bucket for storing the resulting logs and video clips, connected using IAM role.

```json
{
	"Version": "2012-10-17",
	"Statement": [
		{ "Effect": "Allow", "Action": "s3:ListBucket",
		  "Resource": "arn:aws:s3:::incor-doom-runs" },
		{ "Effect": "Allow", "Action": ["s3:PutObject", "s3:GetObject"],
		  "Resource": "arn:aws:s3:::incor-doom-runs/*" }
	]
}
```

Testing the virtual screen out.

```bash
$ sudo apt install -y xvfb pulseaudio ffmpeg
$ Xvfb :99 -screen 0 640x480x24 &
$ export DISPLAY=:99
$ pulseaudio --start --exit-idle-time=-1
$ pactl load-module module-null-sink sink_name=virt
$ pactl set-default-sink virt
$ export SDL_AUDIODRIVER=pulseaudio
$ chocolate-doom -iwad $IWAD -playdemo demo1 -window -geometry 640x480 &
$ ffmpeg -y -f x11grab -video_size 640x480 -framerate 35 -i :99 \
	-f pulse -i virt.monitor -c:v libx264 -preset veryfast -crf 23 \
	-pix_fmt yuv420p -c:a aac -t 205 doom.mp4
$ aws s3 cp doom.mp4 s3://incor-doom-runs/
$ aws s3 presign s3://incor-doom-runs/doom.mp4 --expires-in 3600
```

Get the modified code from my github [Incorrectone/chocolate-doom](https://github.com/Incorrectone/chocolate-doom/tree/butterfly). Snippets below.

Now, watching the same doom run is kind of boring, so I thought of changing something every run so there can be a noticeable changing effect. To do this I had to just change the `prndindex` variable in the game.

```c
void P_Ticker (void)
{
	...
	if (paused)
		return;

	{
		static int nudge_tic = -2;
		extern int prndindex;
		if (nudge_tic == -2)
		{
			const char *s = getenv("NUDGE_TIC");
			nudge_tic = s ? atoi(s) : -1;
		}
		if (gametic == nudge_tic)
			prndindex = (prndindex + 1) & 0xff;
	}
...
```
{: file="src/doom/p_tick.c"}

Also to get a definitive idea of change, I added a log of player state and world hash. I chose the environment variable `STATELOG` to tell whether to enable it.

```c
static unsigned int fnv(unsigned int h, int v) // above P_Ticker
{
	unsigned char *b = (unsigned char *) &v;
	int k;
	for (k = 0; k < 4; k++) { h ^= b[k]; h *= 16777619u; }
	return h;
}
...
// end of P_Ticker, after leveltime++
	if (st_log && players[consoleplayer].mo)
	{
		mobj_t *m = players[consoleplayer].mo;
		unsigned int wh = 2166136261u;
		for (th = thinkercap.next; th != &thinkercap; th = th->next)
			if (th->function.acp1 == (actionf_p1) P_MobjThinker)
			{
				mobj_t *mo = (mobj_t *) th;
				wh = fnv(wh, mo->type); wh = fnv(wh, mo->x); wh = fnv(wh, mo->y);
				wh = fnv(wh, mo->z); wh = fnv(wh, mo->health); wh = fnv(wh, mo->tics);
			}
		for (s = 0; s < numsectors; s++)
		{
			wh = fnv(wh, sectors[s].floorheight);
			wh = fnv(wh, sectors[s].ceilingheight);
		}
		fprintf(st_log, "%d %d %d %d %d %u\n", gametic,
		m->x, m->y, (int)m->angle, players[consoleplayer].health, wh);
	}
```
{: file="src/doom/p_tick.c (simplified)"}

Compile:

```bash
$ git clone https://github.com/chocolate-doom/chocolate-doom.git
$ cd chocolate-doom
$ cmake -B build && cmake --build build -j8
```

Trying the nudge out.

```bash
$ STATELOG=a.txt ./build/src/chocolate-doom -iwad $IWAD -timedemo demo1 -nomusic &
$ STATELOG=b.txt NUDGE_TIC=1000 ./build/src/chocolate-doom -iwad $IWAD -timedemo demo1 -nomusic &
$ wait
$ diff a.txt b.txt | head -3
1085,7117c1085,7117
< 1084 -27722261 40583375 16777216 77
# gametic, x, y, angle, health
```

Trying out 32 runs, with nudges from 200 to 7000th game tick every 200 ticks.

```bash
$ export IWAD
$ seq 200 200 7000 | xargs -P8 -I{} sh -c \ # runs 8 at a time
  'STATELOG=n{}.txt NUDGE_TIC={} ./build/src/chocolate-doom -iwad $IWAD \
   -timedemo demo1 -nomusic >/dev/null 2>&1 || true'

$ for t in $(seq 200 200 7000); do
   paste -d' ' a.txt n$t.txt | awk -v t=$t '
    BEGIN { w=-1; p=-1 }
    { if (w<0 && $6!=$12) w=$1
      if (p<0 && ($2!=$8 || $3!=$9 || $4!=$10 || $5!=$11)) p=$1 }
    END { print t, w, p }'
  done | tee gaps2.txt
```

![Butterfly effect with 32 instances](/doom-bucket/butterfly32.png){: .w-50}
_Butterfly effect with 32 instances_

Testing concurrent rendering on a virtual screen with sound at 16 instances the sound was already slightly bugged, and at 32 instances it became very apparent.

```bash
$ for i in $(seq 0 15); do
	x=$(( (i % 4) * 320 )); y=$(( (i / 4) * 200 ))
	printf 'fullscreen 0\nwindow_width 320\nwindow_height 200\nwindow_position
	"%d,%d"\naspect_ratio_correct 0\n' $x $y > cfg$i.cfg
  done
$ for i in $(seq 0 15); do
	if [ $i -eq 0 ]; then N=""; else N="NUDGE_TIC=$((i*400))"; fi
	env $N SDL_AUDIODRIVER=pulseaudio SNDLOG=ev$i.txt ./build/src/chocolate-doom -iwad $IWAD \
	-playdemo demo1 -nomusic -extraconfig cfg$i.cfg > doom$i.log 2>&1 &
  done
$ ffmpeg -y -f x11grab -draw_mouse 0 -video_size 1280x800 -framerate 35 -i :99 \
	-f pulse -i virt.monitor -c:v libx264 -preset veryfast -crf 23 -pix_fmt yuv420p \
	-c:a aac -t 205 grid16.mp4 2>&1 | tee ffmpeg.log
```

The issue here of note was the sound in concurrent runs. They were very distorted, most likely due to CPU timing being a little off when being recorded by ffmpeg. So an alternate route was to create a log of all the sounds played by every game with the time at which they were played. And then recreate the soundtrack afterwards.

Writes one line per requested sound: tic, sound ID, volume (0-127), stereo separation (0-254, 128 = centre),  pitch, wall clock time.

```c
#include <time.h>
#include "d_loop.h"

// in S_StartSound function
	{
		static FILE *snd_log = NULL;
		if (!snd_log && getenv("SNDLOG"))
		{
			snd_log = fopen(getenv("SNDLOG"), "w");
			if (snd_log) setvbuf(snd_log, NULL, _IOLBF, 0);
		}

		if (snd_log)
		{
			struct timespec ts;
			clock_gettime(CLOCK_REALTIME, &ts);
			fprintf(snd_log, "%d %d %d %d %d %.6f\n", gametic, sfx_id, volume, sep, pitch,
			ts.tv_sec + ts.tv_nsec / 1e9);
		}
	}

	// kill old sound
	S_StopSound(origin);
```
{: file="src/doom/s_sound.c"}

This creates a log of sound events with the wall clock times. Which we can merge with video later.

Test with 8 Runs, This should be the same.

```bash
$ pids=""; for i in $(seq 1 8); do
	SDL_VIDEODRIVER=dummy SDL_AUDIODRIVER=dummy SNDLOG=sounds$i.txt \
		./build/src/chocolate-doom -iwad $IWAD -timedemo demo1 -nomusic > run$i.log 2>&1 &
	pids="$pids $!"
  done; wait $pids
$ grep -h timed run*.log
$ md5sum sounds*.txt
```

## the Screen scaling problem

Screen recording real time windows doesn't scale to thousands of tiles and is not in sync.

Using [doomgeneric](https://github.com/ozkl/doomgeneric) with headless and our patches (it also uses chocolate-doom). We can also shrink the renders to smaller resolutions to fit them into a 4K mosaic.

```bash
$ git clone https://github.com/Incorrectone/doomgeneric.git && cd doomgeneric
$ git checkout -b butterfly && git apply butterfly.patch
$ cd doomgeneric && make -f Makefile.headless -j8 # 320x200, gcc -O2, no X11
$ STATELOG=a_dg.txt ./doomgeneric -iwad $IWAD -timedemo demo1 -nosound
$ cmp a_dg.txt ~/chocolate-doom/a.txt # checks if same
```

Also the `c7a.2xlarge` instance was hitting its peak CPU usage making it necessary to change the machine.

![peak cpu usage](/doom-bucket/8coremaxxed.png)
_c7a.2xlarge hitting peak cpu usage_

I originally chose the `c7a.48xlarge`, 192 CPU cores, but AWS did not have spot instances of that size.

![Bezos Loses](/doom-bucket/bezoslost.png)
_Bezos Loses_

Sometimes smaller is just enough.

![c7a.32xlarge](/doom-bucket/128cores.png)
_c7a.32xlarge_

Look at how big it is, truly beautiful.

## the Code and files

Quick scripts to make it all into neat commands.

1. `layout.py` : Calculates a suitable layout.

   | Tiles | Grid | Tile size | Video | Filled |
   | :--- | :--- | :--- | :--- | :--- |
   | 64 | 8×8 | 320×200 (full) | 2846×1600 | 90% |
   | 128 | 12×11 | 314×196 | 3840×2160 | 95% |
   | 256 | 16×16 | 216×135 | 3840×2160 | 90% |
   | 512 | 24×22 | 157×98 | 3840×2160 | 95% |
   | 1,024 | 34×31 | 111×69 | 3840×2160 | 96% |
   | 2,048 | 48×43 | 80×50 | 3840×2160 | 99% |
   | 4,096 | 68×61 | 56×35 | 3840×2160 | 98% |
   | 8,192 | 96×86 | 40×25 | 3840×2160 | ≈100% |

2. `compositor.c` : The compositor is the program that turns thousands of separate Doom copies into one video. Each Doom copy draws its own small picture. The compositor collects those pictures, arranges them in a grid and hands the finished frames to ffmpeg to encode.
3. `mix_events.py` : Mixes sound from all the sound logs into a single wav file.

	```bash
	$ T0=$(grep -m1 -o 'start: [0-9.]*' ffmpeg.log | cut -d' ' -f2)
	$ python3 mix_events.py --wad $IWAD --sounds-c src/doom/sounds.c --out mix.wav --t0 $T0 ev*.txt
	$ ffmpeg -y -i grid32.mp4 -i mix.wav -map 0:v -map 1:a -c:v copy -c:a aac -shortest grid32_events.mp4
	$ ffmpeg -y -i grid32.mp4 -vn -ac 1 -ar 8000 live.wav && python3 align_audio.py live.wav mix.wav
	```

4. `run_mosaic.sh` : Runs all the headless instances with their correct resolutions and layouts and into their own log files. Also merges them.

	```bash
	$ ./run_mosaic.sh TILES STEP NAME [MAXRES]
	```

## the Big runs

Linux has a limit on the number of processes that can be spawned by default, so if running on the local system you might have to check and change the limit to greater than at least 9000. My AWS instance already had the limits raised.

```bash
$ for n in 256 512 1024 2048 4096; do
	step=$(( 7000 / n )); [ $step -lt 1 ] && step=1
	start=$(date +%s)
	./run_mosaic.sh $n $step big$n < /dev/null > big$n.log 2>&1 &
	pid=$!; peak=0
	while kill -0 $pid 2>/dev/null; do
		used=$(free -m | awk '/Mem:/ {print $3}'); [ $used -gt $peak ] && peak=$used; sleep 5
	done
	secs=$(( $(date +%s) - start ))
	echo "$n,$secs,$(awk -v s=$secs 'BEGIN{printf "%.1f", 7116/s}'),$(awk -v m=$peak 'BEGIN{printf "%.1f",
	m/1024}')" \
	| tee -a scaling_c7a48.csv
	aws s3 cp big$n/big$n.mp4 s3://incor-doom-runs/runs/big$n/ # upload each run immediately
	aws s3 cp big$n/visible_split.txt s3://incor-doom-runs/runs/big$n/
	aws s3 cp big$n.log s3://incor-doom-runs/runs/big$n/
	rm -f big$n/video.mp4 big$n/audio.wav
  done
```

The 8,192 instance run was executed separately right after, seeing if the spot instance had enough time left before any interruption.

| Tiles | Seconds | Frames/s | Peak memory |
| :--- | :--- | :--- | :--- |
| 128 | 240 | 29.6 | n/a (reconstructed) |
| 256 | 270 | 26.4 | 15.4 GB |
| 512 | 276 | 25.8 | 18.1 GB |
| 1,024 | 311 | 22.9 | 21.9 GB |
| 2,048 | 381 | 18.7 | 29.4 GB |
| 4,096 | 481 | 14.8 | 45.4 GB |
| 8,192 | 783 | 9.1 | 77 GB (from sysmon) |
{: style="margin: 0 auto;" }

*Table: Scaling results across tile counts*
{: .text-center .text-muted .small }

## the Results

![Run time and throughput versus number of Doom instances](/doom-bucket/scaling.png)
_Run time and throughput versus number of Doom instances_

Left: wall clock time for the whole demo grows slowly; right: game time simulated per real second
keeps climbing, reaching about 2,100× real time at 8,192 instances.

![Memory and CPU over the 1,024-8,192 runs](/doom-bucket/machine.png)
_Memory and CPU over the 1,024-8,192 runs_

Memory stepped up with each run but peaked at 77 GB of 246 GB. While rendering, CPU averaged
21%, 29%, 41% and 53%; the brief 100% spikes are thousands of processes being launched.

![Seconds until each of 7,038 nudges became visible (8,192 run), with player-state splits from the sweep](/doom-bucket/butterfly8192.png)
_Seconds until each of 7,038 nudges became visible (8,192 run), with player-state splits from the sweep_

Each blue dot is one Doom. Sloping runs that fall to zero are waves: every nudge in a run is revealed
by the same on-screen event, so the delays shrink linearly up to it. Median 0.4 s; 79% visible within a
second; the player's own state (orange) takes far longer. 78 tiles never changed before the end and
1,075 had nudges scheduled after the demo finished.

## the End

As you can see there was plenty of processing power and RAM left. I could have gone through with 16,384 runs as well, but I was on a spot instance and I was not too keen on pushing my luck with the sudden interruption. So I left it there. All the cost came out to be around $4.23 including the development instance and the final spot instance.