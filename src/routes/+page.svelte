<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import mqtt, { type MqttClient } from 'mqtt';
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
    AlertTriangle
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
  const TOPIC = 'wokwi/sensor/alvin/data';

  let client: MqttClient | null = null;
  let status = $state<'connecting' | 'connected' | 'disconnected'>('connecting');
  let lastUpdated = $state<string>('-');
  let data = $state<SensorData | null>(null);

  onMount(() => {
    client = mqtt.connect(BROKER_WS, {
      clientId: `web_${Math.random().toString(16).slice(2, 10)}`,
      clean: true,
      reconnectPeriod: 2000
    });

    client.on('connect', () => {
      status = 'connected';
      client?.subscribe(TOPIC, (err) => {
        if (err) console.error('Sub error:', err);
      });
    });

    client.on('message', (topic, payload) => {
      if (topic !== TOPIC) return;
      try {
        data = JSON.parse(payload.toString());
        lastUpdated = new Date().toLocaleTimeString();
      } catch (e) {
        console.error('Parse error:', e);
      }
    });

    client.on('offline', () => {
      status = 'disconnected';
    });

    client.on('error', (err) => {
      console.error('MQTT error:', err);
      status = 'disconnected';
    });
  });

  onDestroy(() => {
    client?.end();
  });
</script>

<svelte:head>
  <title>IoT Realtime Monitor</title>
</svelte:head>

<div class="min-h-screen bg-background text-foreground">
  <div class="mx-auto max-w-5xl px-4 py-8">
    <header class="mb-8 flex flex-col gap-4 sm:flex-row sm:items-center sm:justify-between">
      <div>
        <div class="flex items-center gap-2">
          <Activity class="size-6 text-primary" />
          <h1 class="text-2xl font-bold tracking-tight">IoT Realtime Monitor</h1>
        </div>
        <p class="text-xs text-muted-foreground mt-1">
          Topic: <code class="rounded bg-muted px-1.5 py-0.5 text-xs text-foreground font-mono">{TOPIC}</code>
        </p>
      </div>

      <div class="flex flex-wrap items-center gap-2">
        <Badge
          variant={status === 'connected' ? 'default' : status === 'connecting' ? 'secondary' : 'destructive'}
          class="flex items-center gap-1.5 px-3 py-1 text-xs"
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
      <Card.Root class="border-dashed p-12 text-center">
        <Card.Content class="flex flex-col items-center justify-center gap-3">
          <Activity class="size-8 animate-pulse text-muted-foreground" />
          <div class="text-sm font-medium text-muted-foreground">
            Waiting for sensor data from Wokwi / HiveMQ...
          </div>
        </Card.Content>
      </Card.Root>
    {:else}
      <div class="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-4">
        <!-- Temperature -->
        <Card.Root class={data.tempBad ? 'border-destructive bg-destructive/10' : ''}>
          <Card.Header class="flex flex-row items-center justify-between pb-2">
            <Card.Title class="text-sm font-medium text-muted-foreground">Temperature</Card.Title>
            <Thermometer class="size-4 {data.tempBad ? 'text-destructive' : 'text-muted-foreground'}" />
          </Card.Header>
          <Card.Content>
            <div class="text-3xl font-bold tracking-tight">
              {data.temp.toFixed(1)} <span class="text-base font-normal text-muted-foreground">°C</span>
            </div>
            {#if data.tempBad}
              <div class="mt-2 flex items-center gap-1 text-xs font-semibold text-destructive">
                <AlertTriangle class="size-3" />
                <span>Out of range (24 - 32 °C)</span>
              </div>
            {/if}
          </Card.Content>
        </Card.Root>

        <!-- Humidity -->
        <Card.Root class={data.humdBad ? 'border-destructive bg-destructive/10' : ''}>
          <Card.Header class="flex flex-row items-center justify-between pb-2">
            <Card.Title class="text-sm font-medium text-muted-foreground">Humidity</Card.Title>
            <Droplets class="size-4 {data.humdBad ? 'text-destructive' : 'text-muted-foreground'}" />
          </Card.Header>
          <Card.Content>
            <div class="text-3xl font-bold tracking-tight">
              {data.humd.toFixed(1)} <span class="text-base font-normal text-muted-foreground">%</span>
            </div>
            {#if data.humdBad}
              <div class="mt-2 flex items-center gap-1 text-xs font-semibold text-destructive">
                <AlertTriangle class="size-3" />
                <span>Abnormal humidity (70 - 85 %)</span>
              </div>
            {/if}
          </Card.Content>
        </Card.Root>

        <!-- Soil Moisture -->
        <Card.Root>
          <Card.Header class="flex flex-row items-center justify-between pb-2">
            <Card.Title class="text-sm font-medium text-muted-foreground">Soil Moisture</Card.Title>
            <Gauge class="size-4 text-muted-foreground" />
          </Card.Header>
          <Card.Content>
            <div class="text-3xl font-bold tracking-tight">
              {data.moist} <span class="text-base font-normal text-muted-foreground">%</span>
            </div>
            <p class="mt-2 text-xs text-muted-foreground">
              ADC Raw: {data.soil}
            </p>
          </Card.Content>
        </Card.Root>

        <!-- Gas / CO Ratio -->
        <Card.Root class={data.airBad ? 'border-destructive bg-destructive/10' : ''}>
          <Card.Header class="flex flex-row items-center justify-between pb-2">
            <Card.Title class="text-sm font-medium text-muted-foreground">Air Ratio (Rs/R0)</Card.Title>
            <Wind class="size-4 {data.airBad ? 'text-destructive' : 'text-muted-foreground'}" />
          </Card.Header>
          <Card.Content>
            <div class="text-3xl font-bold tracking-tight">
              {data.ratio.toFixed(3)}
            </div>
            <p class="mt-2 text-xs text-muted-foreground">
              Rs: {data.rs.toFixed(2)} kΩ | Raw: {data.gas}
            </p>
            {#if data.airBad}
              <div class="mt-2 flex items-center gap-1 text-xs font-semibold text-destructive">
                <AlertTriangle class="size-3" />
                <span>High gas concentration</span>
              </div>
            {/if}
          </Card.Content>
        </Card.Root>
      </div>
    {/if}
  </div>
</div>
