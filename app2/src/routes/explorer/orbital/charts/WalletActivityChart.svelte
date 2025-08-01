<script lang="ts">
import Card from "$lib/components/ui/Card.svelte"
import Skeleton from "$lib/components/ui/Skeleton.svelte"
import { Option, pipe } from "effect"
import type { ActiveWalletRates, WalletStats } from "../types"

interface Props {
  activeSenders: Option.Option<WalletStats[]>
  activeReceivers: Option.Option<WalletStats[]>
  activeSendersTimeScale: Option.Option<Record<string, WalletStats[]>>
  activeReceiversTimeScale: Option.Option<Record<string, WalletStats[]>>
  activeWalletRates: Option.Option<ActiveWalletRates>
}

let {
  activeSenders,
  activeReceivers,
  activeSendersTimeScale,
  activeReceiversTimeScale,
  activeWalletRates,
}: Props = $props()

// Local item count configuration
const itemCounts = [
  { value: 3, label: "3" },
  { value: 5, label: "5" },
  { value: 7, label: "7" },
  { value: 10, label: "10" },
]

// State management
let selectedTimeFrame = $state("5m")
let selectedItemCount = $state(5) // Default to 5 items

// Time frame configuration
const timeFrames = [
  { key: "5m", label: "5m", field: "LastMin", desc: "last 5 minutes" },
  { key: "1h", label: "1h", field: "LastHour", desc: "last hour" },
  { key: "1d", label: "1d", field: "LastDay", desc: "last day" },
  { key: "7d", label: "7d", field: "Last7d", desc: "last 7 days" },
  { key: "14d", label: "14d", field: "Last14d", desc: "last 14 days" },
  { key: "30d", label: "30d", field: "Last30d", desc: "last 30 days" },
] as const

// Derived state using Effect Option patterns
const currentSenders = $derived.by(() => {
  return pipe(
    activeSendersTimeScale,
    Option.flatMap((timeScaleData) => {
      if (timeScaleData[selectedTimeFrame] && timeScaleData[selectedTimeFrame].length > 0) {
        return Option.some(timeScaleData[selectedTimeFrame])
      }
      return activeSenders
    }),
    Option.getOrElse(() => []),
    (data) => data.slice(0, selectedItemCount),
  )
})

const currentReceivers = $derived.by(() => {
  return pipe(
    activeReceiversTimeScale,
    Option.flatMap((timeScaleData) => {
      if (timeScaleData[selectedTimeFrame] && timeScaleData[selectedTimeFrame].length > 0) {
        return Option.some(timeScaleData[selectedTimeFrame])
      }
      return activeReceivers
    }),
    Option.getOrElse(() => []),
    (data) => data.slice(0, selectedItemCount),
  )
})

const chartData = $derived({
  activeSenders: currentSenders,
  activeReceivers: currentReceivers,
  activeWalletRates: pipe(activeWalletRates, Option.getOrElse(() => undefined)),
})

const hasData = $derived(currentSenders.length > 0 || currentReceivers.length > 0)
const isLoading = $derived(
  !hasData && Option.isNone(activeSenders) && Option.isNone(activeReceivers),
)

// Get total transfer count for percentage calculation
const totalTransfersForTimeframe = $derived(() => {
  const senderSum = currentSenders.reduce((sum, sender) => sum + sender.count, 0)
  const receiverSum = currentReceivers.reduce((sum, receiver) => sum + receiver.count, 0)
  return Math.max(senderSum, receiverSum, 1)
})

// Utility functions
function formatAddress(address: string): string {
  if (address.length <= 16) {
    return address
  }
  return `${address.slice(0, 8)}...${address.slice(-6)}`
}

function formatCount(count: number): string {
  if (count === 0) {
    return "0"
  }
  if (count >= 1000000) {
    return `${(count / 1000000).toFixed(1)}M`
  }
  if (count >= 1000) {
    return `${(count / 1000).toFixed(1)}K`
  }
  return count.toString()
}

function getWalletCounts() {
  const timeFrame = timeFrames.find(tf => tf.key === selectedTimeFrame)
  if (!timeFrame || !chartData.activeWalletRates) {
    return { senders: 0, receivers: 0, total: 0 }
  }

  const field = timeFrame.field
  return {
    senders: chartData.activeWalletRates[`senders${field}` as keyof ActiveWalletRates] as number
      || 0,
    receivers: chartData.activeWalletRates[`receivers${field}` as keyof ActiveWalletRates] as number
      || 0,
    total: chartData.activeWalletRates[`total${field}` as keyof ActiveWalletRates] as number || 0,
  }
}

function getPercentageOfTotal(count: number): number {
  return Math.round((count / totalTransfersForTimeframe()) * 100)
}

// Debug logging in development
$effect(() => {
  if (import.meta.env.DEV) {
    console.log("WalletActivityChart data:", {
      hasData,
      isLoading,
      currentSendersLength: currentSenders.length,
      currentReceiversLength: currentReceivers.length,
      hasActiveSenders: Option.isSome(activeSenders),
      hasActiveReceivers: Option.isSome(activeReceivers),
      hasActiveWalletRates: Option.isSome(activeWalletRates),
      selectedItemCount: selectedItemCount,
    })
  }
})

const walletCounts = $derived(getWalletCounts())
const selectedTimeFrameInfo = $derived(timeFrames.find(tf => tf.key === selectedTimeFrame))
</script>

<Card class="h-full p-0">
  <div class="flex flex-col h-full font-mono">
    <!-- Terminal Header -->
    <header class="flex items-center justify-between p-2 border-b border-zinc-800">
      <div class="flex items-center space-x-2">
        <span class="text-zinc-500 text-xs">$</span>
        <h3 class="text-xs text-zinc-300 font-semibold">active-wallets</h3>
        {#if selectedTimeFrameInfo}
          <span class="text-zinc-600 text-xs">--tf={selectedTimeFrameInfo.key}</span>
        {/if}
      </div>
      <div class="text-xs text-zinc-500">
        {#if isLoading}
          loading...
        {:else if !hasData}
          no data yet
        {/if}
      </div>
    </header>

    <!-- Controls -->
    <div class="pt-2 px-2">
      <!-- Mobile: Stack vertically, Desktop: Side by side -->
      <div class="flex flex-col sm:flex-row sm:items-center sm:justify-between gap-2 sm:gap-1 mb-1">
        <!-- Time Frame Selector -->
        <div class="flex flex-wrap gap-0.5">
          {#each timeFrames as timeFrame}
            <button
              class="
                px-2 py-1 text-xs font-mono border transition-colors min-h-[32px] {
                selectedTimeFrame === timeFrame.key
                ? 'border-zinc-500 bg-zinc-800 text-zinc-200 font-medium'
                : 'border-zinc-700 bg-zinc-900 text-zinc-400 hover:border-zinc-600 hover:text-zinc-300'
                }
              "
              onclick={() => selectedTimeFrame = timeFrame.key}
            >
              {timeFrame.label}
            </button>
          {/each}
        </div>

        <!-- Item Count Selector -->
        <div class="flex items-center gap-0.5">
          <span class="text-zinc-600 text-xs font-mono">show:</span>
          {#each itemCounts as itemCount}
            <button
              class="
                px-2 py-1 text-xs font-mono border transition-colors min-h-[32px] {
                selectedItemCount === itemCount.value
                ? 'border-zinc-500 bg-zinc-800 text-zinc-200 font-medium'
                : 'border-zinc-700 bg-zinc-900 text-zinc-400 hover:border-zinc-600 hover:text-zinc-300'
                }
              "
              onclick={() => selectedItemCount = itemCount.value}
            >
              {itemCount.label}
            </button>
          {/each}
        </div>
      </div>
    </div>

    <!-- Main Content -->
    <main class="flex-1 flex flex-col p-2">
      {#if isLoading}
        <!-- Loading State -->
        <div class="grid grid-cols-1 md:grid-cols-2 gap-2 flex-1">
          <!-- Top Senders Skeleton -->
          <div>
            <div class="text-xs text-zinc-500 font-mono font-medium mb-1">
              top_senders:
            </div>
            <div class="space-y-0.5">
              {#each Array(selectedItemCount) as _, index}
                <div class="p-1.5 bg-zinc-900 border border-zinc-800 rounded">
                  <div class="flex items-center justify-between mb-0.5">
                    <div class="flex items-center space-x-1">
                      <Skeleton class="w-2 h-2" />
                      <Skeleton class="w-16 h-2" />
                    </div>
                    <Skeleton class="w-8 h-2" />
                  </div>
                  <div class="flex items-center space-x-2">
                    <Skeleton class="flex-1 h-1" />
                    <Skeleton class="w-6 h-2" />
                  </div>
                </div>
              {/each}
            </div>
          </div>

          <!-- Top Receivers Skeleton -->
          <div>
            <div class="text-xs text-zinc-500 font-mono font-medium mb-1">
              top_receivers:
            </div>
            <div class="space-y-0.5">
              {#each Array(selectedItemCount) as _, index}
                <div class="p-1.5 bg-zinc-900 border border-zinc-800 rounded">
                  <div class="flex items-center justify-between mb-0.5">
                    <div class="flex items-center space-x-1">
                      <Skeleton class="w-2 h-2" />
                      <Skeleton class="w-16 h-2" />
                    </div>
                    <Skeleton class="w-8 h-2" />
                  </div>
                  <div class="flex items-center space-x-2">
                    <Skeleton class="flex-1 h-1" />
                    <Skeleton class="w-6 h-2" />
                  </div>
                </div>
              {/each}
            </div>
          </div>
        </div>
      {:else if !hasData}
        <!-- No Data State -->
        <div class="flex-1 flex items-center justify-center">
          <div class="text-center">
            <div class="text-zinc-600 font-mono">no_data</div>
          </div>
        </div>
      {:else}
        <!-- Wallet Data -->
        <div class="grid grid-cols-1 md:grid-cols-2 gap-2 flex-1">
          <!-- Top Senders -->
          <section>
            <div class="text-xs text-zinc-500 font-mono font-medium mb-1">
              top_senders:
            </div>
            <div class="space-y-1 overflow-y-auto">
              {#each chartData.activeSenders.slice(0, selectedItemCount) as sender, index}
                <article class="p-2 sm:p-1.5 bg-zinc-900 border border-zinc-800 rounded">
                  <!-- Sender Header -->
                  <div class="flex items-center justify-between mb-0.5">
                    <div class="flex items-center space-x-1 text-xs">
                      <span class="text-zinc-500">#{index + 1}</span>
                      <span
                        class="text-zinc-300 font-medium"
                        title={sender.address}
                      >
                        {formatAddress(sender.displayAddress || sender.address)}
                      </span>
                    </div>
                    <span class="text-zinc-100 text-xs tabular-nums font-medium">
                      {formatCount(sender.count)}
                    </span>
                  </div>

                  <!-- Progress Bar -->
                  <div class="flex items-center space-x-2">
                    <div class="flex-1 flex w-full h-1.5 sm:h-1">
                      <div
                        class="bg-zinc-300 h-full transition-all duration-300"
                        style="width: {(sender.count / totalTransfersForTimeframe()) * 100}%"
                        title="Count: {sender.count}"
                      >
                      </div>
                      <div
                        class="bg-zinc-800 h-full transition-all duration-300"
                        style="width: {100 - (sender.count / totalTransfersForTimeframe()) * 100}%"
                      >
                      </div>
                    </div>
                    <span class="text-zinc-500 text-xs tabular-nums">
                      {getPercentageOfTotal(sender.count)}%
                    </span>
                  </div>
                </article>
              {/each}

              {#if chartData.activeSenders.length === 0}
                <div class="text-center py-2">
                  <div class="text-zinc-600 text-xs font-mono">no_data</div>
                </div>
              {/if}
            </div>
          </section>

          <!-- Top Receivers -->
          <section>
            <div class="text-xs text-zinc-500 font-mono font-medium mb-1">
              top_receivers:
            </div>
            <div class="space-y-1 overflow-y-auto">
              {#each chartData.activeReceivers.slice(0, selectedItemCount) as receiver, index}
                <article class="p-2 sm:p-1.5 bg-zinc-900 border border-zinc-800 rounded">
                  <!-- Receiver Header -->
                  <div class="flex items-center justify-between mb-0.5">
                    <div class="flex items-center space-x-1 text-xs">
                      <span class="text-zinc-500">#{index + 1}</span>
                      <span
                        class="text-zinc-300 font-medium"
                        title={receiver.address}
                      >
                        {
                          formatAddress(
                            receiver.displayAddress || receiver.address,
                          )
                        }
                      </span>
                    </div>
                    <span class="text-zinc-100 text-xs tabular-nums font-medium">
                      {formatCount(receiver.count)}
                    </span>
                  </div>

                  <!-- Progress Bar -->
                  <div class="flex items-center space-x-2">
                    <div class="flex-1 flex w-full h-1.5 sm:h-1">
                      <div
                        class="bg-zinc-300 h-full transition-all duration-300"
                        style="width: {(receiver.count / totalTransfersForTimeframe()) * 100}%"
                        title="Count: {receiver.count}"
                      >
                      </div>
                      <div
                        class="bg-zinc-800 h-full transition-all duration-300"
                        style="width: {100 - (receiver.count / totalTransfersForTimeframe()) * 100}%"
                      >
                      </div>
                    </div>
                    <span class="text-zinc-500 text-xs tabular-nums">
                      {getPercentageOfTotal(receiver.count)}%
                    </span>
                  </div>
                </article>
              {/each}

              {#if chartData.activeReceivers.length === 0}
                <div class="text-center py-2">
                  <div class="text-zinc-600 text-xs font-mono">no_data</div>
                </div>
              {/if}
            </div>
          </section>
        </div>
      {/if}
    </main>
  </div>
</Card>

<style>
/* Custom scrollbar styling */
.overflow-y-auto::-webkit-scrollbar {
  width: 4px;
}

.overflow-y-auto::-webkit-scrollbar-track {
  background: #27272a;
}

.overflow-y-auto::-webkit-scrollbar-thumb {
  background: #52525b;
  border-radius: 2px;
}

.overflow-y-auto::-webkit-scrollbar-thumb:hover {
  background: #71717a;
}
</style>
