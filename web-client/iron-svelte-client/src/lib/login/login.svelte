<script lang="ts">
    import { currentSession, userInteractionService } from '../../services/session.service';
    import type { UserInteraction } from '../../../static/iron-remote-desktop';
    import type { Session } from '../../models/session';
    import { preConnectionBlob, displayControl, kdcProxyUrl, init } from '../../../static/iron-remote-desktop-rdp';
    import { toast } from '$lib/messages/message-store';
    import { showLogin } from '$lib/login/login-store';
    import { onMount } from 'svelte';
    const params = new URLSearchParams(window.location.search);
    let username = 'Administrator';
    let password = '';
    // gatewayAddress 为当前域名+7171端口
    let gatewayAddress = `ws://${window.location.hostname}:7171/jet/rdp`;
    // let hostname = '10.10.0.3:3389';
    // hosname 从 URL 参数中获取
    let hostname = params.get('hostname') || '';
    let domain = '';
    let authtoken = params.get('authtoken') || '';
    let kdc_proxy_url = '';
    let desktopSize = { width: 1280, height: 720 };
    let pcb = '';
    let pop_up = false;
    let enable_clipboard = true;

    let userInteraction: UserInteraction;

    const initListeners = () => {
        userInteraction.onSessionEvent((event) => {
            if (event.type === 2) {
                console.log('Error event', event.data);

                toast.set({
                    type: 'error',
                    message: typeof event.data !== 'string' ? event.data.backtrace() : event.data,
                });
            } else {
                toast.set({
                    type: 'info',
                    message: typeof event.data === 'string' ? event.data : event.data?.backtrace() ?? 'No info',
                });
            }
        });
    };

    userInteractionService.subscribe((val) => {
        userInteraction = val;
        if (val != null) {
            initListeners();
        }
    });

    const StartSession = async () => {
        if (authtoken === '') {
            const token_server_url = import.meta.env.VITE_IRON_TOKEN_SERVER_URL as string | undefined;
            if (token_server_url === undefined || token_server_url.trim() === '') {
                toast.set({
                    type: 'error',
                    message: 'Token server is not set and no token provided',
                });
                throw new Error('Token server is not set and no token provided');
            }
            try {
                const response = await fetch(`${token_server_url}/forward`, {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json',
                    },
                    body: JSON.stringify({
                        dst_hst: hostname,
                        jet_ap: 'rdp',
                        jet_ttl: 3600,
                        jet_rec: false,
                    }),
                });

                const data = await response.json();
                if (response.ok) {
                    authtoken = data.token;
                } else if (data.error !== undefined) {
                    throw new Error(data.error);
                } else {
                    throw new Error('Unknown error occurred');
                }
            } catch (error) {
                console.error('Error fetching token:', error);
                toast.set({
                    type: 'error',
                    message: 'Error fetching token',
                });
            }
        }

        toast.set({
            type: 'info',
            message: 'Connection in progress...',
        });

        if (pop_up) {
            const data = JSON.stringify({
                username,
                password,
                hostname,
                gatewayAddress,
                domain,
                authtoken,
                desktopSize,
                pcb,
                kdc_proxy_url,
                enable_clipboard,
            });
            const base64Data = btoa(data);
            window.open(
                `/popup-session?data=${base64Data}`,
                '_blank',
                `width=${desktopSize.width},height=${desktopSize.height},resizable=yes,scrollbars=yes,status=yes`,
            );
            return;
        }

        userInteraction.setEnableClipboard(enable_clipboard);

        const configBuilder = userInteraction
            .configBuilder()
            .withUsername(username)
            .withPassword(password)
            .withDestination(hostname)
            .withProxyAddress(gatewayAddress)
            .withServerDomain(domain)
            .withAuthToken(authtoken)
            .withDesktopSize(desktopSize)
            .withExtension(displayControl(false));

        if (pcb !== '') {
            configBuilder.withExtension(preConnectionBlob(pcb));
        }

        if (kdc_proxy_url !== '') {
            configBuilder.withExtension(kdcProxyUrl(kdc_proxy_url));
        }

        const config = configBuilder.build();

        try {
            const session_info = await userInteraction.connect(config);

            toast.set({
                type: 'info',
                message: 'Success',
            });

            const updater = (session: Session): Session => ({
                ...session,
                sessionId: session_info.sessionId,
                desktopSize: session_info.initialDesktopSize,
                active: true,
            });

            currentSession.update(updater);

            showLogin.set(false);
        } catch (err) {
            console.error(`Error occurred: ${err}`);
        }
    };

    onMount(async () => {
        await init('INFO');
    });
</script>

<main class="responsive login-container">
    <div class="login-content">
        <div class="grid">
            <div class="s2" />
            <div class="s8">
                <article class="primary-container">
                    <h5>Login</h5>
                    <div class="medium-space" />
                    <div>
                        <div class="field label border">
                            <input id="username" type="text" bind:value={username} />
                            <label for="username">Username</label>
                        </div>
                        <div class="field label border">
                            <input id="password" type="password" bind:value={password} />
                            <label for="password">Password</label>
                        </div>
                    </div>
                    <nav class="center-align">
                        <button on:click={StartSession}>Login</button>
                    </nav>
                </article>
            </div>
            <div class="s2" />
        </div>
    </div>
</main>

<style>
    @import './login.css';
</style>
