# Изменения в коде по сравнению с первой частью

Вторая игра («Дело о серийном маньяке», PILOTS2.EXE, 1998) работает на следующей версии того же проигрывателя Game Maker, что и первая («По следам полосатого слона», PILOTS.EXE, 1997). Восстановленный код второй части сделан из кода первой (проект pilots_re) переносом всех отличий, найденных при сравнении двух разобранных баз IDA по каждой функции. В этой странице перечислено, что поменялось в коде и почему.

В проигрывателе (без библиотеки C) добавилось 19 функций и изменилось 25:

- новые: `GmCamera_ResetDefaults`, `GmCamera_FollowAndFlush`, `GmDirty_AddWindowRect`, `Host_ScrollWindow`, `GmTyped_Clear`, `GmTyped_Push`, `GmLoad_GraphicsBatch`, `GmCd_Open`, `GmCd_Play`, `GmCd_Command`, `GmCd_Close`, `GmCd_Stop`, `GmAudio_GetSoundVolume`, `GmAudio_GetMusicVolume`, `Native_TypedTextContains`, `Native_PlayCdTrack`, `Native_CdControl`, `Native_CameraFollow`, `Native_SetCameraParams`;
- изменённые: `GmGame_Tick`, `GmRender_Flush`, `GmInput_Dispatch`, `Host_MainWndProc`, `Host_ShowHelp`, `GmLoad_Run`, `GmSave_ReadSettings` и `GmSave_WriteHeader` (размер настроек), `GmSaveGame_Load`, `GmMovie_Begin`, `GmMovie_End`, `Native_GetOption`, `Native_SetOption`, `GmAudio_Init`, `GmAudio_Shutdown`, `GmAudio_StopAndShutdown`, `GmAudio_Restart`, `GmWave_Open`, `GmWave_Submit`, `GmWave_GetVolume`, `GmWave_SetVolume`, `GmMidi_Open`, `GmMidi_Pump`, `GmMidi_GetVolume`, `GmMidi_SetVolume`.

Остальные функции совпадают с первой частью с точностью до адресов (`GmLoad_GameHeader` отличается только проверяемой версией). Тексты сообщений и прочие постоянные данные совпадают; добавились только строки «%d %d %d %d» (параметры камеры) и «cdaudio» (устройство MCI).

## Формат данных (gm.h, data.c)

| Что | Первая часть | Вторая часть |
|-----|--------------|--------------|
| `GM_FILE_VERSION_CURRENT` — версия файла игры и сохранений | 0x80000016 | 0x80000018 |
| `GM_SETTINGS` | 52 байта | 76 байт |
| `g_NativeFunctions` | 57 функций | 62 функции |
| `GM_LOAD_TAG` | теги 0–5 и 0xFF | добавлены `GMTAG_GRAPHICS_BATCH` (6) и `GMTAG_GRAPHICS_BODY` (7) |
| `GM_OPTION_SET` / `GM_OPTION_QUERY` | ввод, звук, музыка, фильмы | добавлены `GMOPT_CD_OFF`/`GMOPT_CD_ON` (8/9) и `GMOPTQ_CD` (4) |

Новые поля `GM_SETTINGS` вставлены между `moviesOn` и `soundChannels` (смещения 0x18–0x2F):

| Поле | Тип | Назначение |
|------|-----|-----------|
| `cdAudioOn` | `bool` | музыка с компакт-диска включена |
| `cdTrack` | `uint8_t` | играющая дорожка компакт-диска, 0xFF — не играет |
| `reserved1A[2]` | | выравнивание |
| `scrollX`, `scrollY` | `int` | сдвиг вида в мире, пикселей |
| `followObjectIndex` | `uint16_t` | запись пула объекта, за которым следит камера; 0xFFFF — не следит |
| `scrollStep` | `uint16_t` | начальный шаг прокрутки за кадр |
| `slowdownDistance` | `uint16_t` | остаток пути, с которого шаг меняется |
| `slowdownStepDelta` | `int16_t` | изменение шага за кадр на последнем участке |
| `followMarginLeft/Right/Top/Bottom` | `uint8_t` | поля следования камеры в тайлах |

Поэтому сохранения первой и второй частей несовместимы между собой (их не пускает и проверка версии).

Новые перечисления: `GM_CD_COMMAND` (остановить, пауза, продолжить, открыть лоток), `GM_CAMERA_PARAMS` (поля следования или скорость), `GM_GFX_BATCH_CMD` (команды пакета графики). Новые глобальные переменные: `g_WaveVolume`, `g_CdDeviceId`, `g_CdOpened`, `s_CdAudioDevice`, `g_SavedSoundVolume`, `g_SavedMusicVolume`, `g_TypedChars`, `s_CameraParamsFormat`. Глобальные `g_ScrollX`/`g_ScrollY` первой части (всегда 0) убраны — сдвиг вида теперь в настройках.

## Камера и прокрутка вида (game.c, objects.c, host.c)

В первой части сдвиг вида учитывался при выводе и вводе, но всегда был 0. Во второй он живёт в `g_Settings.scrollX/scrollY` и меняется камерой.

- `GmRender_Flush` — вычитает сдвиг из прямоугольников и спрайтов (как раньше) и копирует фон со сдвигом: `GmFrame_CopyRect(..., -(scrollX + clipLeft), -(scrollY + clipTop), ...)` вместо `-clipLeft`, `-clipTop`.
- `GmInput_Dispatch` — переводит координаты указателя в координаты мира прибавлением сдвига.
- `GmDirty_AddWindowRect` (новая) — грязный прямоугольник в координатах окна переводится в координаты мира; ею пользуется `Host_MainWndProc` для `WM_PAINT` и смены палитры вместо прямого `GmDirty_Add`.
- `GmCamera_ResetDefaults` (новая) — параметры камеры по умолчанию сразу после чтения записи настроек (кроме загрузки сохранения): сдвиг 0, слежения нет, шаг 16, торможение с 80 пикселей по −1, поля 0.
- `GmCamera_FollowAndFlush` (новая) — конец такта: если камера следит за объектом, сдвигает вид так, чтобы тайл объекта остался в полях следования, шагами с торможением; на каждом шаге `Host_ScrollWindow` и `GmRender_Flush`, в конце — `GmRender_Flush`.
- `Host_ScrollWindow` (новая) — `ScrollWindow` и `UpdateWindow` главного окна.
- `GmGame_Tick` заканчивается вызовом `GmCamera_FollowAndFlush` вместо `GmRender_Flush`.

Подробно — [rendering.md](rendering.md).

## Коды с клавиатуры (game.c, natives.c, load.c)

- `GmTyped_Clear`, `GmTyped_Push` (новые) — буфер 32 последних набранных знаков `g_TypedChars`.
- `GmGame_Tick` кладёт в буфер каждый знак такта, кроме `GMEV_NONE`. Порядок запоминания полей такта тоже чуть сдвинулся: признак «указатель в окне» теперь запоминается первым.
- `GmLoad_Run` очищает буфер в начале загрузки эпизода.
- `Native_TypedTextContains` (№57, новая) — есть ли в буфере строка; при совпадении буфер очищается.

Подробно — [input.md](input.md).

## Музыка с компакт-диска (audio.c, natives.c, save.c, movie.c, load.c)

- `GmCd_Open`, `GmCd_Play`, `GmCd_Command`, `GmCd_Close`, `GmCd_Stop` (новые) — MCI-устройство «cdaudio»: открыть, играть дорожку с начала (формат TMSF), остановить / пауза / продолжить / открыть лоток, закрыть.
- `GmAudio_Init` в конце всегда открывает привод; `GmAudio_Shutdown` первым делом закрывает его.
- `GmAudio_StopAndShutdown` (перед AVI и внешней программой) запоминает текущие громкости (`g_SavedSoundVolume`, `g_SavedMusicVolume`) и останавливает дорожку, не забывая её; `GmAudio_Restart` возвращает громкости и возобновляет дорожку и музыку. Для этого добавлены `GmAudio_GetSoundVolume` и `GmAudio_GetMusicVolume`.
- `GmLoad_Run`, `GmMovie_Begin`, `GmMovie_End` останавливают и забывают дорожку (`GmCd_Stop`).
- `GmSaveGame_Load` оставляет текущими `cdAudioOn` (и, новое, `musicOn`) и после входа в комнату возобновляет сохранённую дорожку.
- `Native_PlayCdTrack` (№58) и `Native_CdControl` (№59) — новые; `Native_SetOption` получил коды 8/9, `Native_GetOption` — код 4.

Подробно — [audio.md](audio.md).

## Громкость (wave.c)

В первой части громкость менялась у устройств (`waveOutSetVolume`, `midiOutSetVolume`) и читалась у них же. Во второй она программная, шкала 0..0x80:

- `GmWave_Open` ставит `g_WaveVolume = 0x80`; `GmWave_GetVolume`/`GmWave_SetVolume` читают и пишут её; `GmWave_Submit` при нулевой громкости не отправляет буфер. Промежуточные значения звуки не ослабляют.
- `GmMidi_Open` ставит `g_MidiVolume = 0x80` (в первой части — 1); `GmMidi_GetVolume` возвращает её; `GmMidi_Pump` больше не глушит ноты при нуле, а умножает второй байт данных нот на `g_MidiVolume / 0x80`.

В проекте первой части (pilots_re) громкость тоже сделана программной, но как отступление от оригинала; здесь же код просто повторяет вторую часть.

## Пакет графики (load.c)

`GmLoad_GraphicsBatch` (новая) разбирает тег 6: записи графики эпизода идут одним сжатым потоком команд. `GmLoad_Run` вызывает её по `GMTAG_GRAPHICS_BATCH`. Подробно — [resources.md](resources.md).

## Справка (host.c, natives.c)

`Host_ShowHelp` понимает два новых вида строки: «!» — поиск (`HELP_FINDER`) на Windows NT, иначе содержание; строка с цифрой в начале — раздел по номеру (`HELP_CONTEXT`). Подробно — [ui.md](ui.md).

## Камера из скриптов (natives.c)

`Native_CameraFollow` (№60) и `Native_SetCameraParams` (№61) — новые.

## Имена файлов (host.c, build.bat, pilots2.rc)

Файл игры — `PILOTS2.EXE`, сохранения — `PILOTS2<слот>.SAV` и `PILOTS20.230`, справка — `PILOTS2.HLP` (все имена выводятся из имени EXE, как и в первой части). Собранный EXE — `PILOTS2_RE.EXE`, значок взят из PILOTS2.EXE.

## Что в библиотеке C

Изменения в библиотеке Digital Mars C (раскрутка SEH, деление 64-разрядных, таблицы инициализации) — только раскладка адресов; на восстановленный код они не влияют, он пользуется стандартной библиотекой MSVC.

## Отступления от оригинала, добавленные во второй части

Кроме отступлений, унаследованных от pilots_re (см. README в корне проекта):

- `Native_SetCameraParams` обнуляет все четыре числа; оригинал не обнулял первое, и при пустой строке в поле попадал мусор со стека.
- `GmCd_Play` без открытого привода возвращает `false`; в оригинале итог не определён (регистр не задавался).
- Структуры параметров MCI перед вызовом обнуляются (в оригинале неиспользуемые поля содержали мусор со стека).
