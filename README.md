# Discord-Quest-Badge-Free-For-Korean
퀘스트를 완료했습니다. 디스코드 배지 무료로 얻기
1. 키보드로 [Windows + R] 키 입력 2. %appdata%\discord 입력 3. [settings.json]을 우클릭해서 메모장으로 열기 4. 싹 다 지우고 다음과 같이 복붙
2.  { "DANGEROUS_ENABLE_DEVTOOLS_ONLY_ENABLE_IF_YOU_KNOW_WHAT_YOURE_DOING": true } 저장하고 디코 껐다 키기 (디코 프로그램 자체를 재부팅해야함)
5. 아무 서버 들어간뒤 음성방에 아무 창이나 스트리밍 한 뒤 부계정이나 친구 불러서 2인 이상 있는 상태로
6. Ctrl + Shift + I 눌러서 개발자 도구 연다음에 콘솔로 들어간다음 밑에 코드 복붙. 복붙하려하면 안된다고 뜰텐데 안된다고 "뜨면" allow pasting 입력 후 복붙
7. 엔터 누르고 15분 기다림.
코드 :
let wpRequire;
window.webpackChunkdiscord_app.push([[ Math.random() ], {}, (req) => { wpRequire = req; }]);

let ApplicationStreamingStore = Object.values(wpRequire.c).find(x => x?.exports?.default?.getStreamerActiveStreamMetadata).exports.default;
let RunningGameStore = Object.values(wpRequire.c).find(x => x?.exports?.default?.getRunningGames).exports.default;
let QuestsStore = Object.values(wpRequire.c).find(x => x?.exports?.default?.getQuest).exports.default;
let ExperimentStore = Object.values(wpRequire.c).find(x => x?.exports?.default?.getGuildExperiments).exports.default;
let FluxDispatcher = Object.values(wpRequire.c).find(x => x?.exports?.default?.flushWaitQueue).exports.default;
let api = Object.values(wpRequire.c).find(x => x?.exports?.getAPIBaseURL).exports;

let quest = [...QuestsStore.quests.values()].find(x => x.userStatus?.enrolledAt && !x.userStatus?.completedAt && new Date(x.config.expiresAt).getTime() > Date.now())
let isApp = navigator.userAgent.includes("Electron/")
if(!isApp) {
	console.log("This no longer works in browser. Use the desktop app!")
} else if(!quest) {
	console.log("You don't have any uncompleted quests!")
} else {
	const pid = Math.floor(Math.random() * 30000) + 1000
	
	let applicationId, applicationName, secondsNeeded, secondsDone, canPlay
	if(quest.config.configVersion === 1) {
		applicationId = quest.config.applicationId
		applicationName = quest.config.applicationName
		secondsNeeded = quest.config.streamDurationRequirementMinutes * 60
		secondsDone = quest.userStatus?.streamProgressSeconds ?? 0
		canPlay = quest.config.variants.includes(2)
	} else if(quest.config.configVersion === 2) {
		applicationId = quest.config.application.id
		applicationName = quest.config.application.name
		canPlay = ExperimentStore.getUserExperimentBucket("2024-04_quest_playtime_task") > 0 && quest.config.taskConfig.tasks["PLAY_ON_DESKTOP"]
		const taskName = canPlay ? "PLAY_ON_DESKTOP" : "STREAM_ON_DESKTOP"
		secondsNeeded = quest.config.taskConfig.tasks[taskName].target
		secondsDone = quest.userStatus?.progress?.[taskName]?.value ?? 0
	}

	if(canPlay) {
		api.HTTP.get({url: `/applications/public?application_ids=${applicationId}`}).then(res => {
			const appData = res.body[0]
			const exeName = appData.executables.find(x => x.os === "win32").name.replace(">","")
			
			const games = RunningGameStore.getRunningGames()
			const fakeGame = {
				cmdLine: `C:\\Program Files\\${appData.name}\\${exeName}`,
				exeName,
				exePath: `c:/program files/${appData.name.toLowerCase()}/${exeName}`,
				hidden: false,
				isLauncher: false,
				id: applicationId,
				name: appData.name,
				pid: pid,
				pidPath: [pid],
				processName: appData.name,
				start: Date.now(),
			}
			games.push(fakeGame)
			FluxDispatcher.dispatch({type: "RUNNING_GAMES_CHANGE", removed: [], added: [fakeGame], games: games})
			
			let fn = data => {
				let progress = data.userStatus.streamProgressSeconds
				console.log(`Quest progress: ${progress}/${secondsNeeded}`)
				
				if(progress >= secondsNeeded) {
					console.log("Quest completed!")
					
					const idx = games.indexOf(fakeGame)
					if(idx > -1) {
						games.splice(idx, 1)
						FluxDispatcher.dispatch({type: "RUNNING_GAMES_CHANGE", removed: [fakeGame], added: [], games: []})
					}
					FluxDispatcher.unsubscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn)
				}
			}
			FluxDispatcher.subscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn)
			
			console.log(`Spoofed your game to ${applicationName}. Wait for ${Math.ceil((secondsNeeded - secondsDone) / 60)} more minutes.`)
		})
	} else {
		let realFunc = ApplicationStreamingStore.getStreamerActiveStreamMetadata
		ApplicationStreamingStore.getStreamerActiveStreamMetadata = () => ({
			id: applicationId,
			pid,
			sourceName: null
		})
		
		let fn = data => {
			let progress = data.userStatus.streamProgressSeconds
			console.log(`Quest progress: ${progress}/${secondsNeeded}`)
			
			if(progress >= secondsNeeded) {
				console.log("Quest completed!")
				
				ApplicationStreamingStore.getStreamerActiveStreamMetadata = realFunc
				FluxDispatcher.unsubscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn)
			}
		}
		FluxDispatcher.subscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn)
		
		console.log(`Spoofed your stream to ${applicationName}. Stream any window in vc for ${Math.ceil((secondsNeeded - secondsDone) / 60)} more minutes.`)
		console.log("Remember that you need at least 1 other person to be in the vc!")
	}
}
