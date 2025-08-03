<script lang="ts">
    import { onMount } from 'svelte';
    import { setCurrentSessionActive, userInteractionService } from '../../services/session.service';
    import { showLogin } from '$lib/login/login-store';
    import type { UserInteraction } from '../../../static/iron-remote-desktop';
    import { Backend } from '../../../static/iron-remote-desktop-rdp';

    let uiService: UserInteraction;
    let cursorOverrideActive = false;
    let showDebugPanel = false;

    userInteractionService.subscribe((uis) => {
        if (uis != null) {
            uiService = uis;
            uiService.onSessionEvent((event) => {
                if (event.type === 0) {
                    uiService.setVisibility(true);
                } else if (event.type === 1) {
                    setCurrentSessionActive(false);
                    showLogin.set(true);
                }
            });
        }
    });

    function onUnicodeModeChange(e: MouseEvent) {
        if (e.target == null) {
            return;
        }

        let element = e.target as HTMLInputElement;

        if (element == null) {
            return;
        }

        uiService.setKeyboardUnicodeMode(element.checked);
    }

    function toggleCursorKind() {
        if (cursorOverrideActive) {
            uiService.setCursorStyleOverride(null);
        } else {
            uiService.setCursorStyleOverride('url("crosshair.png") 7 7, default');
        }

        cursorOverrideActive = !cursorOverrideActive;
    }

    onMount(async () => {
        let el = document.querySelector('iron-remote-desktop');

        if (el == null) {
            throw '`iron-remote-desktop` element not found';
        }

        el.addEventListener('ready', (e) => {
            const event = e as CustomEvent;
            userInteractionService.set(event.detail.irgUserInteraction);
        });
    });
</script>

<div style="display: flex; height: 100%; flex-direction: column; background-color: #2e2e2e;" class:hideall={$showLogin}>
    <iron-remote-desktop verbose="true" scale="fit" flexcenter="true" module={Backend} />
</div>

<style>
    .hideall {
        display: none !important;
    }
</style>
