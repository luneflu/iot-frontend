<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import { Tween } from 'svelte/motion';
  import { cubicOut } from 'svelte/easing';
  import mqtt, { type MqttClient } from 'mqtt';
  import { asset } from '$app/paths';
  import { DotLottieSvelte } from '@lottiefiles/dotlottie-svelte';
  import { toast } from 'svelte-sonner';
  import * as Card from '$lib/components/ui/card/index.js';
  import { Badge } from '$lib/components/ui/badge/index.js';
  import {
    Thermometer,
    Droplets,
    Wind,
    Gauge,
    Activity,
    Wifi,
    WifiOff,
    AlertTriangle,
    RotateCcw
  } from '@lucide/svelte';

  interface SensorData {
    gas: number;
    rs: number;
    ratio: number;
    soil: number;
    moist: number;
    humd: number;
    temp: number;
    tempBad: boolean;
    humdBad: boolean;
    airBad: boolean;
  }

  const BROKER_WS = 'wss://broker.hivemq.com:8884/mqtt';
  let topic = $state('wokwi/sensor/alvin/data');

  let client: MqttClient | null = null;
  let status = $state<'connecting' | 'connected' | 'disconnected'>('connecting');
  let lastUpdated = $state<string>('-');
  let data = $state<SensorData | null>(null);

  // Stats
  let maxTemp = $state<number>(-Infinity);
  let minTemp = $state<number>(Infinity);

  // Smooth tweened values (Svelte 5 runes)
  const tempTween = new Tween(0, { duration: 500, easing: cubicOut });
  const humdTween = new Tween(0, { duration: 500, easing: cubicOut });
  const moistTween = new Tween(0, { duration: 500, easing: cubicOut });
  const ratioTween = new Tween(1, { duration: 500, easing: cubicOut });

  // Previous alert flags for one-shot toasts
  let prevTempBad = false;
  let prevHumdBad = false;
  let prevAirBad = false;

  function connectMqtt() {
    if (client) client.end();
    status = 'connecting';

    client = mqtt.connect(BROKER_WS, {
      clientId: `web_${Math.random().toString(16).slice(2, 10)}`,
      clean: true,
      reconnectPeriod: 2000
    });

    client.on('connect', () => {
      status = 'connected';
      toast.success('Connected to HiveMQ Broker', { description: `Subscribed: ${topic}` });
      client?.subscribe(topic, (err) => {
        if (err) toast.error('Subscription error', { description: err.message });
      });
    });

    client.on('message', (t, payload) => {
      if (t !== topic) return;
      try {
        const next: SensorData = JSON.parse(payload.toString());
        data = next;
        lastUpdated = new Date().toLocaleTimeString();

        // Target smooth animations
        tempTween.target = next.temp;
        humdTween.target = next.humd;
        moistTween.target = next.moist;
        ratioTween.target = next.ratio;

        // Min/max tracking
        if (next.temp > maxTemp) maxTemp = next.temp;
        if (next.temp < minTemp) minTemp = next.temp;

        // Alerts (trigger toast only on false -> true transition)
        if (next.tempBad && !prevTempBad) {
          toast.error('High Temperature Alert!', {
            description: `Temp: ${next.temp.toFixed(1)} °C. Exceeds safe limit (24-32 °C).`
          });
        }
        if (next.humdBad && !prevHumdBad) {
          toast.warning('Abnormal Humidity!', {
            description: `Humidity: ${next.humd.toFixed(1)} %. Expected (70-85 %).`
          });
        }
        if (next.airBad && !prevAirBad) {
          toast.error('Hazardous Air / Gas Alert!', {
            description: `Rs/R0: ${next.ratio.toFixed(3)}. Elevated CO/Gas level!`
          });
        }

        prevTempBad = next.tempBad;
        prevHumdBad = next.humdBad;
        prevAirBad = next.airBad;
      } catch (e) {
        console.error('Payload parse error:', e);
      }
    });

    client.on('offline', () => {
      status = 'disconnected';
    });

    client.on('error', (err) => {
      console.error('MQTT error:', err);
      status = 'disconnected';
    });
  }

  function resetStats() {
    if (data) {
      maxTemp = data.temp;
      minTemp = data.temp;
    } else {
      maxTemp = -Infinity;
      minTemp = Infinity;
    }
    toast.info('Peak records reset');
  }

  onMount(() => {
    connectMqtt();
  });

  onDestroy(() => {
    client?.end();
  });

  const hasAlert = $derived(Boolean(data?.tempBad || data?.humdBad || data?.airBad));
</script>

<svelte:head>
  <title>IoT Realtime Monitor</title>
</svelte:head>

<div class="min-h-screen bg-background text-foreground selection:bg-primary/20">
  <div class="mx-auto max-w-5xl px-4 py-8">
    <!-- Header -->
    <header class="mb-8 flex flex-col gap-4 sm:flex-row sm:items-center sm:justify-between border-b border-border/60 pb-6">
      <div class="flex items-center gap-4">
        <!-- Lottie status badge -->
        <div class="size-14 shrink-0 rounded-2xl bg-card border border-border/80 flex items-center justify-center overflow-hidden shadow-sm">
          {#if hasAlert}
            <DotLottieSvelte
              src={asset('/lottie/alert.json')}
              autoplay
              loop
              style="width: 48px; height: 48px;"
            />
          {:else}
            <DotLottieSvelte
              src={asset('/lottie/radar.json')}
              autoplay
              loop
              style="width: 48px; height: 48px;"
            />
          {/if}
        </div>

        <div>
          <div class="flex items-center gap-2">
            <h1 class="text-2xl font-bold tracking-tight">IoT Realtime Monitor</h1>
            {#if hasAlert}
              <Badge variant="destructive" class="animate-pulse text-[10px] uppercase tracking-wider">
                Alert Active
              </Badge>
            {/if}
          </div>
          <div class="flex items-center gap-2 mt-1 text-xs text-muted-foreground">
            <span>Topic:</span>
            <code class="rounded bg-muted px-1.5 py-0.5 text-xs text-foreground font-mono">{topic}</code>
          </div>
        </div>
      </div>

      <div class="flex flex-wrap items-center gap-2 sm:self-center">
        <Badge
          variant={status === 'connected' ? 'default' : status === 'connecting' ? 'secondary' : 'destructive'}
          class="flex items-center gap-1.5 px-3 py-1 text-xs shadow-xs"
        >
          {#if status === 'connected'}
            <Wifi class="size-3.5" />
          {:else}
            <WifiOff class="size-3.5" />
          {/if}
          <span class="capitalize">{status}</span>
        </Badge>
        <span class="text-xs text-muted-foreground">Updated: {lastUpdated}</span>
      </div>
    </header>

    {#if !data}
      <!-- Empty state with pulse -->
      <Card.Root class="border-dashed p-12 text-center bg-card/50">
        <Card.Content class="flex flex-col items-center justify-center gap-3">
          <div class="size-20">
            <DotLottieSvelte
              src={asset('/lottie/radar.json')}
              autoplay
              loop
              style="width: 100%; height: 100%;"
            />
          </div>
          <div class="text-base font-semibold text-foreground">Listening on MQTT Topic</div>
          <p class="text-xs text-muted-foreground max-w-sm">
            Make sure your Wokwi simulation or ESP32 is running and publishing JSON to
            <code class="text-primary">{topic}</code>
          </p>
        </Card.Content>
      </Card.Root>
    {:else}
      <!-- Metric Cards Grid -->
      <div class="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-4">
        <!-- Temperature Card -->
        <Card.Root class="relative transition-all duration-300 {data.tempBad ? 'border-destructive bg-destructive/10 shadow-destructive/20 shadow-md' : 'hover:border-foreground/20'}">
          <Card.Header class="flex flex-row items-center justify-between pb-2">
            <Card.Title class="text-xs font-semibold uppercase tracking-wider text-muted-foreground">Temperature</Card.Title>
            <div class="p-2 rounded-lg {data.tempBad ? 'bg-destructive/20 text-destructive' : 'bg-muted text-foreground'}">
              <Thermometer class="size-4" />
            </div>
          </Card.Header>
          <Card.Content>
            <div class="text-3xl font-extrabold tracking-tight">
              {tempTween.current.toFixed(1)} <span class="text-sm font-medium text-muted-foreground">°C</span>
            </div>

            <!-- Peak tracking row -->
            <div class="mt-3 flex items-center justify-between text-[11px] text-muted-foreground border-t border-border/50 pt-2">
              <span>Min: <strong class="text-foreground">{minTemp.toFixed(1)}°</strong></span>
              <span>Max: <strong class="text-foreground">{maxTemp.toFixed(1)}°</strong></span>
              <button
                onclick={resetStats}
                class="hover:text-foreground text-muted-foreground transition-colors"
                title="Reset peak records"
              >
                <RotateCcw class="size-3" />
              </button>
            </div>

            {#if data.tempBad}
              <div class="mt-2 flex items-center gap-1.5 text-xs font-semibold text-destructive">
                <AlertTriangle class="size-3.5 shrink-0" />
                <span>Out of threshold (24 - 32 °C)</span>
              </div>
            {/if}
          </Card.Content>
        </Card.Root>

        <!-- Humidity Card -->
        <Card.Root class="relative transition-all duration-300 {data.humdBad ? 'border-destructive bg-destructive/10 shadow-destructive/20 shadow-md' : 'hover:border-foreground/20'}">
          <Card.Header class="flex flex-row items-center justify-between pb-2">
            <Card.Title class="text-xs font-semibold uppercase tracking-wider text-muted-foreground">Humidity</Card.Title>
            <div class="p-2 rounded-lg {data.humdBad ? 'bg-destructive/20 text-destructive' : 'bg-muted text-foreground'}">
              <Droplets class="size-4" />
            </div>
          </Card.Header>
          <Card.Content>
            <div class="text-3xl font-extrabold tracking-tight">
              {humdTween.current.toFixed(1)} <span class="text-sm font-medium text-muted-foreground">%</span>
            </div>

            <!-- Linear Progress bar -->
            <div class="mt-3 w-full bg-muted rounded-full h-1.5 overflow-hidden">
              <div
                class="h-full transition-all duration-300 {data.humdBad ? 'bg-destructive' : 'bg-primary'}"
                style="width: {Math.min(100, Math.max(0, humdTween.current))}%"
              ></div>
            </div>

            <p class="mt-2 text-[11px] text-muted-foreground">
              Normal range: 70% – 85%
            </p>

            {#if data.humdBad}
              <div class="mt-2 flex items-center gap-1.5 text-xs font-semibold text-destructive">
                <AlertTriangle class="size-3.5 shrink-0" />
                <span>Abnormal level</span>
              </div>
            {/if}
          </Card.Content>
        </Card.Root>

        <!-- Soil Moisture Card (with Radial Gauge) -->
        <Card.Root class="hover:border-foreground/20 transition-all duration-300">
          <Card.Header class="flex flex-row items-center justify-between pb-2">
            <Card.Title class="text-xs font-semibold uppercase tracking-wider text-muted-foreground">Soil Moisture</Card.Title>
            <div class="p-2 rounded-lg bg-muted text-foreground">
              <Gauge class="size-4" />
            </div>
          </Card.Header>
          <Card.Content class="flex items-center justify-between">
            <div>
              <div class="text-3xl font-extrabold tracking-tight">
                {Math.round(moistTween.current)} <span class="text-sm font-medium text-muted-foreground">%</span>
              </div>
              <p class="mt-1 text-[11px] text-muted-foreground">
                Raw ADC: {data.soil}
              </p>
            </div>

            <!-- Circular SVG Gauge -->
            <div class="relative size-14 shrink-0 flex items-center justify-center">
              <svg class="size-full -rotate-90" viewBox="0 0 36 36">
                <!-- Track -->
                <path
                  class="text-muted stroke-current"
                  stroke-width="3.5"
                  fill="none"
                  d="M18 2.0845 a 15.9155 15.9155 0 0 1 0 31.831 a 15.9155 15.9155 0 0 1 0 -31.831"
                />
                <!-- Progress -->
                <path
                  class="stroke-current transition-all duration-300 text-primary"
                  stroke-width="3.5"
                  stroke-linecap="round"
                  fill="none"
                  stroke-dasharray="{moistTween.current}, 100"
                  d="M18 2.0845 a 15.9155 15.9155 0 0 1 0 31.831 a 15.9155 15.9155 0 0 1 0 -31.831"
                />
              </svg>
              <span class="absolute text-[10px] font-bold text-muted-foreground">
                {Math.round(moistTween.current)}%
              </span>
            </div>
          </Card.Content>
        </Card.Root>

        <!-- Air / MQ9 Gas Card -->
        <Card.Root class="relative transition-all duration-300 {data.airBad ? 'border-destructive bg-destructive/10 shadow-destructive/20 shadow-md' : 'hover:border-foreground/20'}">
          <Card.Header class="flex flex-row items-center justify-between pb-2">
            <Card.Title class="text-xs font-semibold uppercase tracking-wider text-muted-foreground">Air Ratio (Rs/R0)</Card.Title>
            <div class="p-2 rounded-lg {data.airBad ? 'bg-destructive/20 text-destructive' : 'bg-muted text-foreground'}">
              <Wind class="size-4" />
            </div>
          </Card.Header>
          <Card.Content>
            <div class="text-3xl font-extrabold tracking-tight">
              {ratioTween.current.toFixed(3)}
            </div>
            <p class="mt-1 text-[11px] text-muted-foreground">
              Rs: {data.rs.toFixed(2)} kΩ · Raw: {data.gas}
            </p>
            {#if data.airBad}
              <div class="mt-2 flex items-center gap-1.5 text-xs font-semibold text-destructive">
                <AlertTriangle class="size-3.5 shrink-0" />
                <span>CO Gas Warning (&le; 0.70)</span>
              </div>
            {/if}
          </Card.Content>
        </Card.Root>
      </div>
    {/if}
  </div>
</div>
