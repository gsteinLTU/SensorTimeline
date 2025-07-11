<script lang="ts">
    import { browser } from "$app/environment";
    import Peer, { type DataConnection } from "peerjs";
    import CollectStep from "$lib/components/CollectStep.svelte";
    import TrainStep from "$lib/components/TrainStep.svelte";
    import TestStep from "$lib/components/TestStep.svelte";
    import { LocalStore } from "$lib/localStore";
    import { onMount } from "svelte";
    
    let peer: Peer | null = $state.raw(null);
    let peerId: string | null = $state(null);
    let peerStatus: string | null = $state(null);
    let connection: DataConnection | null = $state.raw(null);
    let accelerometerData: {x: number, y: number, z: number, timestamp: number} | null = $state(null);
    let dataHistory: Array<{x: number, y: number, z: number, timestamp: number}> = $state([]);
    let isReceivingData: boolean = $state(false);

    // Input source selection
    let inputSource: 'webrtc' | 'microbit' = $state('microbit');

    // Recording state
    let isRecording: boolean = $state(false);
    let recordingSensorData: Array<{x: number, y: number, z: number, timestamp: number}> = [];

    // LocalStore for recordings
    let recordingsStore: LocalStore<Array<{
        id: string;
        startTime: number;
        endTime: number;
        videoBlob: Blob;
        sensorData: Array<{x: number, y: number, z: number, timestamp: number}>;
        duration: number;
    }>> | null = $state.raw(null);

    onMount(async () => {
        // Initialize LocalStore for recordings
        recordingsStore = await LocalStore.create<Array<{
            id: string;
            startTime: number;
            endTime: number;
            videoBlob: Blob;
            sensorData: Array<{x: number, y: number, z: number, timestamp: number}>;
            duration: number;
        }>>("saved-recordings", []);

        // Load and set recordings state
        const loaded = loadRecordings();
        recordings = loaded;
        console.log('Recordings store initialized:', recordingsStore);
    });

    // Helper: Convert Blob to base64 string
    async function blobToBase64(blob: Blob): Promise<string> {
        return new Promise((resolve, reject) => {
            const reader = new FileReader();
            reader.onloadend = () => resolve(reader.result as string);
            reader.onerror = reject;
            reader.readAsDataURL(blob);
        });
    }

    // Helper: Convert base64 string to Blob
    function base64ToBlob(base64: string, mimeType: string): Blob {
        const byteString = atob(base64.split(',')[1]);
        const ab = new ArrayBuffer(byteString.length);
        const ia = new Uint8Array(ab);
        for (let i = 0; i < byteString.length; i++) {
            ia[i] = byteString.charCodeAt(i);
        }
        return new Blob([ab], { type: mimeType });
    }

    // Patch: Load recordings from LocalStore and rehydrate videoBlob
    function loadRecordings(): Array<any> {
        if (!recordingsStore) {
            return [];
        }

        const raw = recordingsStore.get() || [];

        console.log(`Loaded ${raw.length} recording(s) from store:`, raw);

        return raw.map((rec: any) => {
            if (rec.videoBlob && typeof rec.videoBlob === 'object' && rec.videoBlob.base64 && rec.videoBlob.type) {
                return {
                    ...rec,
                    videoBlob: base64ToBlob(rec.videoBlob.base64, rec.videoBlob.type)
                };
            }
            return rec;
        });
    }

    let recordings: Array<{
        id: string;
        startTime: number;
        endTime: number;
        videoBlob: Blob;
        sensorData: Array<{x: number, y: number, z: number, timestamp: number}>;
        duration: number;
    }> = $state([]);

    function handleIncomingData(rawData: any) {
        try {
            const data = typeof rawData === 'string' ? JSON.parse(rawData) : rawData;
            
            if (data.type === 'accelerometer' && data.data) {
                accelerometerData = data.data;
                dataHistory = [...dataHistory.slice(-99), data.data]; // Keep last 100 readings
                isReceivingData = true;
                
                // If recording, also add to recording sensor data
                if (isRecording) {
                    recordingSensorData = [...recordingSensorData, data.data];
                }
            } else if (data.type === 'heartbeat') {
                console.log('Desktop: Received heartbeat');
            }
        } catch (error) {
            console.error('Desktop: Error parsing incoming data:', error);
        }
    }

    function clearDataHistory() {
        dataHistory = [];
        accelerometerData = null;
        isReceivingData = false;
    }

    // Initialize PeerJS when WebRTC mode is selected
    $effect(() => {
        if (browser && inputSource === 'webrtc' && !peer) {
            const newPeer = new Peer();
            peer = newPeer;

            newPeer.on('open', id => {
                console.log('Peer ID:', id);
                peerStatus = 'Connected';
                peerId = id;
            });

            newPeer.on("connection", (conn) => {
                console.log('Incoming connection from:', conn.peer);
                connection = conn;
                
                conn.on("data", (data) => {
                    console.log('Received data:', data);
                    handleIncomingData(data);
                });
                
                conn.on("open", () => {
                    console.log('Incoming connection opened with:', conn.peer);
                    conn.send(JSON.stringify({
                        type: 'welcome',
                        message: 'Connected to desktop'
                    }));
                });
                
                conn.on('close', () => {
                    console.log('Incoming connection closed');
                    clearDataHistory();
                    connection = null;
                });
                
                conn.on('error', (err) => {
                    console.error('Incoming connection error:', err);
                });
            });
            
            newPeer.on('close', () => {
                console.log('Connection closed');
                peerStatus = 'Disconnected';
            });

            newPeer.on('error', (err) => {
                console.error('PeerJS error:', err);
            });
        } else if (inputSource !== 'webrtc' && peer) {
            // Clean up PeerJS when switching away from WebRTC mode
            if (connection) {
                connection.close();
                connection = null;
            }
            peer.destroy();
            peer = null;
            peerId = null;
            peerStatus = null;
        }
    });

    // Reactive statement to handle input source changes
    $effect(() => {
        // Clear data when switching input sources
        clearDataHistory();
        console.log('Input source changed to:', inputSource);
    });

    let useMockMicroBit = $state(false);

    // Check for ?mockmicrobit=1 in the URL
    if (browser) {
        const params = new URLSearchParams(window.location.search);
        useMockMicroBit = params.get('mockmicrobit') === '1';
    }

    // Step-based workflow state
    type Step = 'collect' | 'train' | 'test';
    let step: Step = $state('collect');

    // Sync step with query param on mount
    onMount(() => {
        if (browser) {
            const params = new URLSearchParams(window.location.search);
            const urlStep = params.get('step');
            if (urlStep === 'collect' || urlStep === 'train' || urlStep === 'test') {
                step = urlStep;
            }
        }
    });

    // Update query param when step changes
    $effect(() => {
        if (browser) {
            const params = new URLSearchParams(window.location.search);
            if (params.get('step') !== step) {
                params.set('step', step);
                const newUrl = `${window.location.pathname}?${params.toString()}`;
                window.history.replaceState({}, '', newUrl);
            }
        }
    });
</script>

<div class="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100 p-8">
    <div class="max-w-2xl mx-auto">
        <div class="bg-white rounded-2xl shadow-xl p-8">
            <h1 class="text-4xl font-bold text-gray-900 mb-8 text-center">
                <span class="bg-gradient-to-r from-blue-600 to-indigo-600 bg-clip-text text-transparent">
                    Sensor Timeline - Desktop Monitor
                </span>
            </h1>

            <!-- Step Content -->
            {#if step === 'collect'}
                <CollectStep stepForward={() => (step = 'train')} />
            {:else if step === 'train'}
                <TrainStep
                    stepBack={() => (step = 'collect')}
                    stepForward={() => (step = 'test')}
                />
            {:else if step === 'test'}
                <TestStep
                    stepBack={() => (step = 'train')}
                />
            {/if}
        </div>
        <!-- Footer -->
        <div class="mt-8 text-center">
            <p class="text-gray-600 text-sm">
                Real-time sensor data streaming with 
                <span class="text-blue-600 font-semibold">SvelteKit</span> + 
                <span class="text-blue-600 font-semibold">PeerJS</span> + 
                <span class="text-blue-600 font-semibold">Device Motion API</span> + 
                <span class="text-purple-600 font-semibold">micro:bit Bluetooth</span>
            </p>
        </div>
    </div>
</div>
