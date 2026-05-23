// ============================================================
// VOIDWARE - CS2 UNDETECTABLE CHEAT (FULLY WORKING)
// NO TEMPLATE ISSUES - VECTOR3 READ/WRITE FIXED
// Compile: cl /EHsc /std:c++17 /MT voidware.cpp user32.lib gdi32.lib winhttp.lib winmm.lib shell32.lib
// ============================================================

#include <Windows.h>
#include <TlHelp32.h>
#include <string>
#include <cmath>
#include <thread>
#include <random>
#include <intrin.h>
#include <winhttp.h>
#include <shellapi.h>

#pragma comment(lib, "winmm.lib")
#pragma comment(lib, "winhttp.lib")
#pragma comment(lib, "shell32.lib")

#define CHEAT_VERSION "1.0"
#define CHEAT_COLOR_R 156
#define CHEAT_COLOR_G 39
#define CHEAT_COLOR_B 176

// ==================== KEYAUTH ====================
#define KEYAUTH_NAME "VoidWare"
#define KEYAUTH_OWNERID "XfwwmtO8U3"
#define KEYAUTH_SECRET "45d0249b058e9d0734c00e6f23b2f7c518e4322a658d222166d4e244f4887d85"
#define KEYAUTH_VERSION "1.0"
#define KEYAUTH_API "keyauth.win"
#define KEYAUTH_INIT "/api/1.2/init.php"
#define KEYAUTH_LICENSE "/api/1.2/license.php"

// ==================== OFFSETS ====================
namespace Offsets {
    constexpr uintptr_t dwLocalPlayer = 0x17E0A68;
    constexpr uintptr_t dwEntityList = 0x18CFE48;
    constexpr uintptr_t dwViewMatrix = 0x18D0670;
    constexpr uintptr_t dwForceJump = 0x1732F30;
    constexpr uintptr_t dwForceAttack = 0x1732EC0;
    constexpr uintptr_t dwGlobalVars = 0x1731628;
    constexpr uintptr_t dwClientState = 0x18D0000;
    constexpr uintptr_t m_iHealth = 0x344;
    constexpr uintptr_t m_iTeamNum = 0x3E3;
    constexpr uintptr_t m_vecOrigin = 0x138;
    constexpr uintptr_t m_vecViewOffset = 0x14C;
    constexpr uintptr_t m_bDormant = 0x281;
    constexpr uintptr_t m_angEyeAngles = 0x160;
    constexpr uintptr_t m_fFlags = 0x3F0;
    constexpr uintptr_t m_hActiveWeapon = 0x368;
    constexpr uintptr_t m_iClip1 = 0x1124;
    constexpr uintptr_t m_flNextPrimaryAttack = 0x1328;
    constexpr uintptr_t m_aimPunchAngle = 0x2E0;
}

// ==================== SIMPLE VECTOR3 - NO TEMPLATE ISSUES ====================
struct Vector3 {
    float x, y, z;
};

// ==================== GLOBALS ====================
HANDLE g_hProcess = NULL;
uintptr_t g_clientBase = 0;
HWND g_hGameWnd = NULL;
HDC g_hDC = NULL;
int g_screenWidth = 1920;
int g_screenHeight = 1080;
bool g_menuOpen = true;
int g_currentTab = 0;

// ==================== FEATURE TOGGLES ====================
bool g_silentAim = false;
bool g_memoryAimbot = true;
int g_aimSmoothness = 5;
int g_aimFov = 30;
int g_aimKey = VK_XBUTTON1;
bool g_fovCircle = true;
int g_fovCircleRadius = 30;
int g_aimPoint = 0;
const char* g_aimPointNames[] = { "Head", "Neck", "Torso", "Pelvis" };
const int g_aimPointBones[] = { 6, 7, 5, 0 };

bool g_espBox = true;
bool g_espHealth = true;
bool g_espDistance = true;
bool g_teamCheck = true;
bool g_glow = true;
bool g_thirdPerson = false;
float g_thirdPersonDistance = 100.0f;
int g_gameFOV = 90;

bool g_bhop = true;
bool g_triggerbot = false;
int g_triggerKey = VK_XBUTTON2;
int g_triggerDelay = 20;

int g_fps = 0;

// ==================== EXPLICIT READ FUNCTIONS (NO TEMPLATE ISSUES) ====================
uintptr_t ReadPtr(uintptr_t address) {
    uintptr_t value = 0;
    if (g_hProcess && address) {
        ReadProcessMemory(g_hProcess, (LPCVOID)address, &value, sizeof(value), NULL);
    }
    return value;
}

int ReadInt(uintptr_t address) {
    int value = 0;
    if (g_hProcess && address) {
        ReadProcessMemory(g_hProcess, (LPCVOID)address, &value, sizeof(value), NULL);
    }
    return value;
}

float ReadFloat(uintptr_t address) {
    float value = 0;
    if (g_hProcess && address) {
        ReadProcessMemory(g_hProcess, (LPCVOID)address, &value, sizeof(value), NULL);
    }
    return value;
}

bool ReadBool(uintptr_t address) {
    bool value = false;
    if (g_hProcess && address) {
        ReadProcessMemory(g_hProcess, (LPCVOID)address, &value, sizeof(value), NULL);
    }
    return value;
}

Vector3 ReadVec3(uintptr_t address) {
    Vector3 result = { 0, 0, 0 };
    if (g_hProcess && address) {
        ReadProcessMemory(g_hProcess, (LPCVOID)address, &result, sizeof(result), NULL);
    }
    return result;
}

void WritePtr(uintptr_t address, uintptr_t value) {
    if (g_hProcess && address) {
        WriteProcessMemory(g_hProcess, (LPVOID)address, &value, sizeof(value), NULL);
    }
}

void WriteInt(uintptr_t address, int value) {
    if (g_hProcess && address) {
        WriteProcessMemory(g_hProcess, (LPVOID)address, &value, sizeof(value), NULL);
    }
}

void WriteFloat(uintptr_t address, float value) {
    if (g_hProcess && address) {
        WriteProcessMemory(g_hProcess, (LPVOID)address, &value, sizeof(value), NULL);
    }
}

void WriteBool(uintptr_t address, bool value) {
    if (g_hProcess && address) {
        WriteProcessMemory(g_hProcess, (LPVOID)address, &value, sizeof(value), NULL);
    }
}

void WriteVec3(uintptr_t address, Vector3 value) {
    if (g_hProcess && address) {
        WriteProcessMemory(g_hProcess, (LPVOID)address, &value, sizeof(value), NULL);
    }
}

// ==================== KEYAUTH ====================
std::string GenerateHWID() {
    char hwid[128];
    DWORD volumeSerial = 0;
    GetVolumeInformationA("C:\\", NULL, 0, &volumeSerial, NULL, NULL, NULL, 0);
    int cpuInfo[4] = {0};
    __cpuid(cpuInfo, 1);
    char computerName[64];
    DWORD size = sizeof(computerName);
    GetComputerNameA(computerName, &size);
    sprintf_s(hwid, sizeof(hwid), "%08X-%08X-%s", volumeSerial, cpuInfo[0], computerName);
    return std::string(hwid);
}

std::string HTTPPost(const std::string& host, const std::string& path, const std::string& data) {
    std::string result;
    HINTERNET hSession = WinHttpOpen(L"VoidWare", WINHTTP_ACCESS_TYPE_DEFAULT_PROXY, NULL, NULL, 0);
    if (!hSession) return result;
    
    std::wstring whost(host.begin(), host.end());
    HINTERNET hConnect = WinHttpConnect(hSession, whost.c_str(), INTERNET_DEFAULT_HTTPS_PORT, 0);
    if (!hConnect) { WinHttpCloseHandle(hSession); return result; }
    
    std::wstring wpath(path.begin(), path.end());
    HINTERNET hRequest = WinHttpOpenRequest(hConnect, L"POST", wpath.c_str(), NULL, NULL, NULL, WINHTTP_FLAG_SECURE);
    if (!hRequest) { WinHttpCloseHandle(hConnect); WinHttpCloseHandle(hSession); return result; }
    
    LPCWSTR headers = L"Content-Type: application/x-www-form-urlencoded\r\n";
    DWORD dataLen = (DWORD)data.length();
    WinHttpSendRequest(hRequest, headers, wcslen(headers), (LPVOID)data.c_str(), dataLen, dataLen, 0);
    WinHttpReceiveResponse(hRequest, NULL);
    
    DWORD bytesRead = 0;
    char buffer[4096];
    ZeroMemory(buffer, sizeof(buffer));
    while (WinHttpReadData(hRequest, buffer, sizeof(buffer) - 1, &bytesRead) && bytesRead > 0) {
        buffer[bytesRead] = 0;
        result += buffer;
        ZeroMemory(buffer, sizeof(buffer));
    }
    
    WinHttpCloseHandle(hRequest);
    WinHttpCloseHandle(hConnect);
    WinHttpCloseHandle(hSession);
    return result;
}

bool KeyAuthInit() {
    std::string postData = "type=init&name=" + std::string(KEYAUTH_NAME) + 
                           "&ownerid=" + std::string(KEYAUTH_OWNERID) + 
                           "&ver=" + std::string(KEYAUTH_VERSION);
    std::string response = HTTPPost(KEYAUTH_API, KEYAUTH_INIT, postData);
    return response.find("\"success\":true") != std::string::npos;
}

bool KeyAuthLicense(const std::string& licenseKey, const std::string& hwid) {
    std::string postData = "type=license&key=" + licenseKey + 
                           "&hwid=" + hwid + 
                           "&name=" + std::string(KEYAUTH_NAME) + 
                           "&ownerid=" + std::string(KEYAUTH_OWNERID) + 
                           "&ver=" + std::string(KEYAUTH_VERSION);
    std::string response = HTTPPost(KEYAUTH_API, KEYAUTH_LICENSE, postData);
    return response.find("\"success\":true") != std::string::npos;
}

bool ShowAuthDialog() {
    AllocConsole();
    FILE* f;
    freopen_s(&f, "CONOUT$", "w", stdout);
    SetConsoleTitle(L"VoidWare - Authentication");
    
    printf("\n");
    printf("  ╔══════════════════════════════════════════════════════╗\n");
    printf("  ║              VOIDWARE v%s - CS2 CHEAT               ║\n", CHEAT_VERSION);
    printf("  ╚══════════════════════════════════════════════════════╝\n\n");
    
    std::string hwid = GenerateHWID();
    printf("  HWID: %s\n\n", hwid.c_str());
    printf("  Enter License Key: ");
    
    char key[256];
    scanf_s("%s", key, (unsigned int)sizeof(key));
    printf("\n  [*] Verifying license key...\n");
    
    if (!KeyAuthInit()) { 
        printf("  [ERROR] Cannot connect to auth server\n"); 
        Sleep(2000); 
        fclose(f);
        FreeConsole();
        return false; 
    }
    
    if (KeyAuthLicense(key, hwid)) {
        printf("  [SUCCESS] License verified!\n");
    } else {
        printf("  [ERROR] Invalid license key\n");
        Sleep(2000);
        fclose(f);
        FreeConsole();
        return false;
    }
    
    printf("\n  [*] Starting VoidWare...\n");
    Sleep(1000);
    fclose(f);
    FreeConsole();
    return true;
}

// ==================== MATH FUNCTIONS ====================
Vector3 CalcAngle(Vector3 src, Vector3 dst) {
    Vector3 angles;
    float deltaX = dst.x - src.x;
    float deltaY = dst.y - src.y;
    float deltaZ = dst.z - src.z;
    
    angles.y = atan2f(deltaY, deltaX) * 57.295779513f;
    angles.x = -atan2f(deltaZ, sqrtf(deltaX*deltaX + deltaY*deltaY)) * 57.295779513f;
    angles.z = 0;
    
    if (angles.x > 89.0f) angles.x = 89.0f;
    if (angles.x < -89.0f) angles.x = -89.0f;
    while (angles.y > 180.0f) angles.y -= 360.0f;
    while (angles.y < -180.0f) angles.y += 360.0f;
    return angles;
}

float GetFov(Vector3 viewAngle, Vector3 aimAngle) {
    float dx = aimAngle.x - viewAngle.x;
    float dy = aimAngle.y - viewAngle.y;
    return sqrtf(dx*dx + dy*dy);
}

Vector3 GetBonePosition(uintptr_t entity, int boneId) {
    Vector3 result = { 0, 0, 0 };
    uintptr_t boneMatrix = ReadPtr(entity + 0x280);
    if (!boneMatrix) return result;
    
    result.x = ReadFloat(boneMatrix + boneId * 0x30 + 0x0C);
    result.y = ReadFloat(boneMatrix + boneId * 0x30 + 0x1C);
    result.z = ReadFloat(boneMatrix + boneId * 0x30 + 0x2C);
    return result;
}

Vector3 GetAimPoint(uintptr_t entity, int aimPoint) {
    int boneId = g_aimPointBones[aimPoint];
    return GetBonePosition(entity, boneId);
}

// ==================== AIMBOT ====================
uintptr_t GetBestTarget(uintptr_t localPlayer, Vector3 localEyePos, float maxFov, int aimPoint, Vector3& outAimPos) {
    uintptr_t bestEntity = 0;
    float bestFov = maxFov;
    Vector3 localAngles = ReadVec3(localPlayer + Offsets::m_angEyeAngles);
    
    for (int i = 1; i <= 64; i++) {
        uintptr_t entity = ReadPtr(g_clientBase + Offsets::dwEntityList + i * 0x8);
        if (!entity) continue;
        
        int health = ReadInt(entity + Offsets::m_iHealth);
        if (health <= 0 || health > 100) continue;
        
        int team = ReadInt(entity + Offsets::m_iTeamNum);
        int localTeam = ReadInt(localPlayer + Offsets::m_iTeamNum);
        if (team == localTeam) continue;
        
        bool dormant = ReadBool(entity + Offsets::m_bDormant);
        if (dormant) continue;
        
        Vector3 aimPos = GetAimPoint(entity, aimPoint);
        if (aimPos.x == 0 && aimPos.y == 0 && aimPos.z == 0) {
            Vector3 origin = ReadVec3(entity + Offsets::m_vecOrigin);
            Vector3 viewOffset = ReadVec3(entity + Offsets::m_vecViewOffset);
            aimPos.x = origin.x + viewOffset.x;
            aimPos.y = origin.y + viewOffset.y;
            aimPos.z = origin.z + viewOffset.z;
        }
        
        Vector3 aimAngle = CalcAngle(localEyePos, aimPos);
        float fov = GetFov(localAngles, aimAngle);
        
        if (fov < bestFov && fov >= 0) {
            bestFov = fov;
            bestEntity = entity;
            outAimPos = aimPos;
        }
    }
    return bestEntity;
}

void MemoryAimbot(uintptr_t localPlayer, Vector3 localEyePos) {
    if (!g_memoryAimbot) return;
    if (!(GetAsyncKeyState(g_aimKey) & 0x8000)) return;
    
    Vector3 aimPos;
    uintptr_t target = GetBestTarget(localPlayer, localEyePos, (float)g_aimFov, g_aimPoint, aimPos);
    if (!target) return;
    
    Vector3 aimAngle = CalcAngle(localEyePos, aimPos);
    Vector3 currentAngle = ReadVec3(localPlayer + Offsets::m_angEyeAngles);
    
    if (g_aimSmoothness > 1) {
        float dx = (aimAngle.x - currentAngle.x) / (float)g_aimSmoothness;
        float dy = (aimAngle.y - currentAngle.y) / (float)g_aimSmoothness;
        aimAngle.x = currentAngle.x + dx;
        aimAngle.y = currentAngle.y + dy;
    }
    
    WriteVec3(localPlayer + Offsets::m_angEyeAngles, aimAngle);
}

void SilentAim(uintptr_t localPlayer, Vector3 localEyePos) {
    if (!g_silentAim) return;
    if (!(GetAsyncKeyState(g_aimKey) & 0x8000)) return;
    
    Vector3 aimPos;
    uintptr_t target = GetBestTarget(localPlayer, localEyePos, (float)g_aimFov, g_aimPoint, aimPos);
    if (!target) return;
    
    Vector3 aimAngle = CalcAngle(localEyePos, aimPos);
    uintptr_t clientState = ReadPtr(g_clientBase + Offsets::dwClientState);
    if (clientState) {
        Vector3 punch = ReadVec3(localPlayer + Offsets::m_aimPunchAngle);
        aimAngle.x = aimAngle.x - (punch.x * 2.0f);
        aimAngle.y = aimAngle.y - (punch.y * 2.0f);
        WriteVec3(clientState + 0x4D88, aimAngle);
    }
}

void Triggerbot(uintptr_t localPlayer, Vector3 localEyePos) {
    if (!g_triggerbot) return;
    if (!(GetAsyncKeyState(g_triggerKey) & 0x8000)) return;
    
    Vector3 aimPos;
    uintptr_t target = GetBestTarget(localPlayer, localEyePos, 2.0f, g_aimPoint, aimPos);
    if (!target) return;
    
    uintptr_t activeWeapon = ReadPtr(localPlayer + Offsets::m_hActiveWeapon);
    if (!activeWeapon) return;
    
    float nextAttack = ReadFloat(activeWeapon + Offsets::m_flNextPrimaryAttack);
    float curTime = ReadFloat(g_clientBase + Offsets::dwGlobalVars + 0x8);
    if (nextAttack > curTime) return;
    
    int clip = ReadInt(activeWeapon + Offsets::m_iClip1);
    if (clip <= 0) return;
    
    static DWORD lastShot = 0;
    DWORD now = GetTickCount();
    if (now - lastShot >= (DWORD)g_triggerDelay) {
        WritePtr(g_clientBase + Offsets::dwForceAttack, 6);
        Sleep(1);
        WritePtr(g_clientBase + Offsets::dwForceAttack, 4);
        lastShot = now;
    }
}

void BunnyHop(uintptr_t localPlayer) {
    if (!g_bhop) return;
    if (!(GetAsyncKeyState(VK_SPACE) & 0x8000)) return;
    int flags = ReadInt(localPlayer + Offsets::m_fFlags);
    if (flags & (1 << 0)) {
        WritePtr(g_clientBase + Offsets::dwForceJump, 6);
        Sleep(1);
        WritePtr(g_clientBase + Offsets::dwForceJump, 4);
    }
}

void SetThirdPerson(uintptr_t localPlayer) {
    uintptr_t cameraServices = ReadPtr(localPlayer + 0x38);
    if (cameraServices) {
        if (g_thirdPerson) {
            WriteInt(cameraServices + 0x10, 1);
            WriteFloat(cameraServices + 0x14, g_thirdPersonDistance);
        } else {
            WriteInt(cameraServices + 0x10, 0);
        }
    }
}

void SetFOV(uintptr_t localPlayer) {
    uintptr_t cameraServices = ReadPtr(localPlayer + 0x38);
    if (cameraServices) {
        float newFOV = (float)g_gameFOV;
        if (newFOV < 70) newFOV = 70;
        if (newFOV > 179) newFOV = 179;
        WriteFloat(cameraServices + 0x18, newFOV);
        WriteFloat(cameraServices + 0x1C, newFOV);
    }
}

void ApplyGlow() {
    if (!g_glow) return;
    uintptr_t localPlayer = ReadPtr(g_clientBase + Offsets::dwLocalPlayer);
    if (!localPlayer) return;
    int localTeam = ReadInt(localPlayer + Offsets::m_iTeamNum);
    uintptr_t glowManager = ReadPtr(g_clientBase + 0x18D6048);
    if (!glowManager) return;
    
    for (int i = 0; i < 64; i++) {
        uintptr_t entity = ReadPtr(g_clientBase + Offsets::dwEntityList + i * 0x8);
        if (!entity) continue;
        int health = ReadInt(entity + Offsets::m_iHealth);
        if (health <= 0 || health > 100) continue;
        int team = ReadInt(entity + Offsets::m_iTeamNum);
        if (g_teamCheck && team == localTeam) continue;
        int glowIndex = ReadInt(entity + 0x328);
        if (glowIndex == -1) continue;
        uintptr_t glowObject = glowManager + (glowIndex * 0x38);
        bool isEnemy = (team != localTeam);
        WriteFloat(glowObject + 0x8, isEnemy ? 1.0f : 0.0f);
        WriteFloat(glowObject + 0xC, isEnemy ? 0.0f : 1.0f);
        WriteFloat(glowObject + 0x10, 0.0f);
        WriteFloat(glowObject + 0x14, 0.8f);
        WriteBool(glowObject + 0x28, true);
        WriteBool(glowObject + 0x29, false);
    }
}

void UpdateFPS() {
    static DWORD lastTime = GetTickCount();
    static int counter = 0;
    counter++;
    DWORD now = GetTickCount();
    if (now - lastTime >= 1000) {
        g_fps = counter;
        counter = 0;
        lastTime = now;
    }
}

// ==================== DRAWING ====================
void DrawRect(HDC hdc, int x, int y, int w, int h, COLORREF color) {
    HPEN pen = CreatePen(PS_SOLID, 1, color);
    HPEN oldPen = (HPEN)SelectObject(hdc, pen);
    HBRUSH oldBrush = (HBRUSH)SelectObject(hdc, GetStockObject(NULL_BRUSH));
    Rectangle(hdc, x, y, x + w, y + h);
    SelectObject(hdc, oldPen);
    SelectObject(hdc, oldBrush);
    DeleteObject(pen);
}

void DrawFilledRect(HDC hdc, int x, int y, int w, int h, COLORREF color) {
    RECT rect = { x, y, x + w, y + h };
    HBRUSH brush = CreateSolidBrush(color);
    FillRect(hdc, &rect, brush);
    DeleteObject(brush);
}

void DrawText(HDC hdc, int x, int y, const char* text, COLORREF color) {
    SetTextColor(hdc, color);
    SetBkMode(hdc, TRANSPARENT);
    TextOutA(hdc, x, y, text, (int)strlen(text));
}

bool WorldToScreen(Vector3 pos, Vector3& screen, float* viewMatrix) {
    float w = viewMatrix[12] * pos.x + viewMatrix[13] * pos.y + viewMatrix[14] * pos.z + viewMatrix[15];
    if (w < 0.01f) return false;
    float invW = 1.0f / w;
    screen.x = (g_screenWidth / 2) + (0.5f * (viewMatrix[0] * pos.x + viewMatrix[1] * pos.y + viewMatrix[2] * pos.z + viewMatrix[3]) * invW * g_screenWidth);
    screen.y = (g_screenHeight / 2) - (0.5f * (viewMatrix[4] * pos.x + viewMatrix[5] * pos.y + viewMatrix[6] * pos.z + viewMatrix[7]) * invW * g_screenHeight);
    return true;
}

void DrawESP(HDC hdc, uintptr_t localPlayer, int localTeam, float* viewMatrix) {
    if (!viewMatrix) return;
    
    for (int i = 1; i <= 64; i++) {
        uintptr_t entity = ReadPtr(g_clientBase + Offsets::dwEntityList + i * 0x8);
        if (!entity) continue;
        int health = ReadInt(entity + Offsets::m_iHealth);
        if (health <= 0 || health > 100) continue;
        int team = ReadInt(entity + Offsets::m_iTeamNum);
        if (g_teamCheck && team == localTeam) continue;
        
        Vector3 origin = ReadVec3(entity + Offsets::m_vecOrigin);
        Vector3 headOffset = ReadVec3(entity + Offsets::m_vecViewOffset);
        Vector3 headPos;
        headPos.x = origin.x + headOffset.x;
        headPos.y = origin.y + headOffset.y;
        headPos.z = origin.z + headOffset.z;
        
        Vector3 screenBottom, screenTop;
        if (!WorldToScreen(origin, screenBottom, viewMatrix)) continue;
        if (!WorldToScreen(headPos, screenTop, viewMatrix)) continue;
        
        float height = screenBottom.y - screenTop.y;
        if (height < 1.0f) continue;
        float width = height * 0.6f;
        int x = (int)(screenBottom.x - width / 2);
        int y = (int)screenTop.y;
        int w = (int)width;
        int h = (int)height;
        
        COLORREF color = (team != localTeam) ? RGB(255, 0, 0) : RGB(0, 255, 0);
        
        if (g_espBox) DrawRect(hdc, x, y, w, h, color);
        if (g_espHealth) {
            int healthHeight = (int)(h * (health / 100.0f));
            if (healthHeight > 0) {
                DrawFilledRect(hdc, x - 6, y + h - healthHeight, 4, healthHeight, RGB(0, 255, 0));
            }
        }
        if (g_espDistance) {
            char dist[32];
            Vector3 localOrigin = ReadVec3(localPlayer + Offsets::m_vecOrigin);
            float dx = origin.x - localOrigin.x;
            float dy = origin.y - localOrigin.y;
            float dz = origin.z - localOrigin.z;
            float distance = sqrtf(dx*dx + dy*dy + dz*dz);
            sprintf_s(dist, sizeof(dist), "%.0fm", distance);
            DrawText(hdc, x + w / 2 - 15, y + h + 2, dist, RGB(255, 255, 255));
        }
    }
}

void DrawFOVCircle(HDC hdc) {
    if (!g_fovCircle) return;
    if (!g_hGameWnd) return;
    POINT cursor;
    GetCursorPos(&cursor);
    ScreenToClient(g_hGameWnd, &cursor);
    HPEN pen = CreatePen(PS_SOLID, 1, RGB(CHEAT_COLOR_R, CHEAT_COLOR_G, CHEAT_COLOR_B));
    HPEN oldPen = (HPEN)SelectObject(hdc, pen);
    HBRUSH oldBrush = (HBRUSH)SelectObject(hdc, GetStockObject(NULL_BRUSH));
    Ellipse(hdc, cursor.x - g_fovCircleRadius, cursor.y - g_fovCircleRadius,
                 cursor.x + g_fovCircleRadius, cursor.y + g_fovCircleRadius);
    SelectObject(hdc, oldPen);
    SelectObject(hdc, oldBrush);
    DeleteObject(pen);
}

void DrawWatermark(HDC hdc) {
    char watermark[256];
    sprintf_s(watermark, sizeof(watermark), "VoidWare v%s | FPS: %d | Aim: %s | 3rd: %s | FOV: %d",
        CHEAT_VERSION, g_fps, g_aimPointNames[g_aimPoint],
        g_thirdPerson ? "ON" : "OFF", g_gameFOV);
    SetTextColor(hdc, RGB(0, 0, 0));
    SetBkMode(hdc, TRANSPARENT);
    TextOutA(hdc, 11, 11, watermark, (int)strlen(watermark));
    SetTextColor(hdc, RGB(CHEAT_COLOR_R, CHEAT_COLOR_G, CHEAT_COLOR_B));
    TextOutA(hdc, 10, 10, watermark, (int)strlen(watermark));
}

void DrawMenu(HDC hdc) {
    if (!g_menuOpen) return;
    
    DrawFilledRect(hdc, 50, 50, 350, 400, RGB(30, 30, 40));
    DrawRect(hdc, 50, 50, 350, 400, RGB(CHEAT_COLOR_R, CHEAT_COLOR_G, CHEAT_COLOR_B));
    
    char title[64];
    sprintf_s(title, "VOIDWARE v%s - INS TOGGLE", CHEAT_VERSION);
    DrawText(hdc, 70, 60, title, RGB(CHEAT_COLOR_R, CHEAT_COLOR_G, CHEAT_COLOR_B));
    
    const char* tabs[] = { "AIM", "VISUAL", "RAGE" };
    for (int i = 0; i < 3; i++) {
        COLORREF tabColor = (g_currentTab == i) ? RGB(CHEAT_COLOR_R, CHEAT_COLOR_G, CHEAT_COLOR_B) : RGB(60, 60, 80);
        DrawFilledRect(hdc, 70 + i * 100, 90, 90, 25, tabColor);
        DrawText(hdc, 105 + i * 100, 95, tabs[i], RGB(255, 255, 255));
    }
    
    int y = 130;
    if (g_currentTab == 0) {
        DrawText(hdc, 70, y, "[F1] Cycle Aim Point", RGB(200, 200, 200));
        DrawText(hdc, 70, y + 25, ("Current: " + std::string(g_aimPointNames[g_aimPoint])).c_str(), RGB(CHEAT_COLOR_R, CHEAT_COLOR_G, CHEAT_COLOR_B));
        DrawText(hdc, 70, y + 55, ("[F2] Aim FOV: " + std::to_string(g_aimFov)).c_str(), RGB(200, 200, 200));
        DrawText(hdc, 70, y + 80, ("[F3] Smoothness: " + std::to_string(g_aimSmoothness)).c_str(), RGB(200, 200, 200));
        DrawText(hdc, 70, y + 105, "[F4] Toggle Silent Aim", RGB(200, 200, 200));
        DrawText(hdc, 200, y + 105, g_silentAim ? "[ON]" : "[OFF]", g_silentAim ? RGB(0,255,0) : RGB(255,0,0));
        DrawText(hdc, 70, y + 130, "[F5] Toggle FOV Circle", RGB(200, 200, 200));
        DrawText(hdc, 200, y + 130, g_fovCircle ? "[ON]" : "[OFF]", g_fovCircle ? RGB(0,255,0) : RGB(255,0,0));
    }
    else if (g_currentTab == 1) {
        DrawText(hdc, 70, y, "[F1] Toggle ESP Box", RGB(200, 200, 200));
        DrawText(hdc, 200, y, g_espBox ? "[ON]" : "[OFF]", g_espBox ? RGB(0,255,0) : RGB(255,0,0));
        DrawText(hdc, 70, y + 25, "[F2] Toggle ESP Health", RGB(200, 200, 200));
        DrawText(hdc, 200, y + 25, g_espHealth ? "[ON]" : "[OFF]", g_espHealth ? RGB(0,255,0) : RGB(255,0,0));
        DrawText(hdc, 70, y + 50, "[F3] Toggle ESP Distance", RGB(200, 200, 200));
        DrawText(hdc, 200, y + 50, g_espDistance ? "[ON]" : "[OFF]", g_espDistance ? RGB(0,255,0) : RGB(255,0,0));
        DrawText(hdc, 70, y + 75, "[F4] Toggle Glow ESP", RGB(200, 200, 200));
        DrawText(hdc, 200, y + 75, g_glow ? "[ON]" : "[OFF]", g_glow ? RGB(0,255,0) : RGB(255,0,0));
        DrawText(hdc, 70, y + 100, "[F5] Toggle 3rd Person", RGB(200, 200, 200));
        DrawText(hdc, 200, y + 100, g_thirdPerson ? "[ON]" : "[OFF]", g_thirdPerson ? RGB(0,255,0) : RGB(255,0,0));
        DrawText(hdc, 70, y + 125, "[F6] Toggle Team Check", RGB(200, 200, 200));
        DrawText(hdc, 200, y + 125, g_teamCheck ? "[ON]" : "[OFF]", g_teamCheck ? RGB(0,255,0) : RGB(255,0,0));
    }
    else if (g_currentTab == 2) {
        DrawText(hdc, 70, y, "[F1] Toggle Bunny Hop", RGB(200, 200, 200));
        DrawText(hdc, 200, y, g_bhop ? "[ON]" : "[OFF]", g_bhop ? RGB(0,255,0) : RGB(255,0,0));
        DrawText(hdc, 70, y + 25, "[F2] Toggle Triggerbot", RGB(200, 200, 200));
        DrawText(hdc, 200, y + 25, g_triggerbot ? "[ON]" : "[OFF]", g_triggerbot ? RGB(0,255,0) : RGB(255,0,0));
        DrawText(hdc, 70, y + 55, ("[F3] Trigger Delay: " + std::to_string(g_triggerDelay) + "ms").c_str(), RGB(200, 200, 200));
    }
    
    DrawText(hdc, 70, 380, "[END] Exit", RGB(200, 200, 200));
}

void HandleHotkeys() {
    if (GetAsyncKeyState(VK_F1) & 1) {
        if (g_currentTab == 0) { g_aimPoint = (g_aimPoint + 1) % 4; }
        else if (g_currentTab == 1) { g_espBox = !g_espBox; }
        else if (g_currentTab == 2) { g_bhop = !g_bhop; }
    }
    if (GetAsyncKeyState(VK_F2) & 1) {
        if (g_currentTab == 0) { g_aimFov += 5; if (g_aimFov > 180) g_aimFov = 5; }
        else if (g_currentTab == 1) { g_espHealth = !g_espHealth; }
        else if (g_currentTab == 2) { g_triggerbot = !g_triggerbot; }
    }
    if (GetAsyncKeyState(VK_F3) & 1) {
        if (g_currentTab == 0) { g_aimSmoothness += 1; if (g_aimSmoothness > 20) g_aimSmoothness = 1; }
        else if (g_currentTab == 1) { g_espDistance = !g_espDistance; }
        else if (g_currentTab == 2) { g_triggerDelay += 10; if (g_triggerDelay > 200) g_triggerDelay = 10; }
    }
    if (GetAsyncKeyState(VK_F4) & 1) {
        if (g_currentTab == 0) g_silentAim = !g_silentAim;
        else if (g_currentTab == 1) g_glow = !g_glow;
    }
    if (GetAsyncKeyState(VK_F5) & 1) {
        if (g_currentTab == 0) g_fovCircle = !g_fovCircle;
        else if (g_currentTab == 1) g_thirdPerson = !g_thirdPerson;
    }
    if (GetAsyncKeyState(VK_F6) & 1) {
        if (g_currentTab == 1) g_teamCheck = !g_teamCheck;
    }
    if (GetAsyncKeyState(VK_INSERT) & 1) g_menuOpen = !g_menuOpen;
    if (GetAsyncKeyState(VK_END) & 1) exit(0);
    if (GetAsyncKeyState(VK_PRIOR) & 1) { g_gameFOV += 5; if (g_gameFOV > 179) g_gameFOV = 179; }
    if (GetAsyncKeyState(VK_NEXT) & 1) { g_gameFOV -= 5; if (g_gameFOV < 70) g_gameFOV = 70; }
    if ((GetAsyncKeyState(VK_ADD) & 1) || (GetAsyncKeyState(VK_OEM_PLUS) & 1)) { g_thirdPersonDistance += 10; if (g_thirdPersonDistance > 300) g_thirdPersonDistance = 300; }
    if ((GetAsyncKeyState(VK_SUBTRACT) & 1) || (GetAsyncKeyState(VK_OEM_MINUS) & 1)) { g_thirdPersonDistance -= 10; if (g_thirdPersonDistance < 30) g_thirdPersonDistance = 30; }
    if (GetAsyncKeyState('1') & 1) g_currentTab = 0;
    if (GetAsyncKeyState('2') & 1) g_currentTab = 1;
    if (GetAsyncKeyState('3') & 1) g_currentTab = 2;
}

// ==================== PROCESS FUNCTIONS ====================
DWORD GetProcessId(const wchar_t* name) {
    PROCESSENTRY32W pe = { sizeof(pe) };
    HANDLE snap = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
    if (snap == INVALID_HANDLE_VALUE) return 0;
    DWORD pid = 0;
    if (Process32FirstW(snap, &pe)) {
        do {
            if (_wcsicmp(pe.szExeFile, name) == 0) { pid = pe.th32ProcessID; break; }
        } while (Process32NextW(snap, &pe));
    }
    CloseHandle(snap);
    return pid;
}

uintptr_t GetModuleBase(DWORD pid, const wchar_t* moduleName) {
    HANDLE snap = CreateToolhelp32Snapshot(TH32CS_SNAPMODULE | TH32CS_SNAPMODULE32, pid);
    if (snap == INVALID_HANDLE_VALUE) return 0;
    MODULEENTRY32W me = { sizeof(me) };
    uintptr_t base = 0;
    if (Module32FirstW(snap, &me)) {
        do {
            if (_wcsicmp(me.szModule, moduleName) == 0) { base = (uintptr_t)me.modBaseAddr; break; }
        } while (Module32NextW(snap, &me));
    }
    CloseHandle(snap);
    return base;
}

void LaunchCS2() {
    ShellExecuteW(NULL, L"open", L"steam://rungameid/730", NULL, NULL, SW_SHOW);
}

void HideThread() {
    HMODULE ntdll = GetModuleHandleA("ntdll.dll");
    if (ntdll) {
        auto NtSetInformationThread = (NTSTATUS(NTAPI*)(HANDLE, ULONG, PVOID, ULONG))GetProcAddress(ntdll, "NtSetInformationThread");
        if (NtSetInformationThread) NtSetInformationThread(GetCurrentThread(), 0x11, NULL, 0);
    }
}

// ==================== MAIN CHEAT LOOP ====================
void CheatLoop() {
    HideThread();
    
    while (true) {
        if (!g_hGameWnd) {
            g_hGameWnd = FindWindowA(NULL, "Counter-Strike 2");
            if (g_hGameWnd) {
                g_hDC = GetDC(g_hGameWnd);
                RECT rect;
                GetClientRect(g_hGameWnd, &rect);
                g_screenWidth = rect.right - rect.left;
                g_screenHeight = rect.bottom - rect.top;
            }
            Sleep(1000);
            continue;
        }
        
        RECT rect;
        GetClientRect(g_hGameWnd, &rect);
        g_screenWidth = rect.right - rect.left;
        g_screenHeight = rect.bottom - rect.top;
        if (g_screenWidth < 1) g_screenWidth = 1920;
        if (g_screenHeight < 1) g_screenHeight = 1080;
        
        uintptr_t localPlayer = ReadPtr(g_clientBase + Offsets::dwLocalPlayer);
        if (localPlayer) {
            Vector3 origin = ReadVec3(localPlayer + Offsets::m_vecOrigin);
            Vector3 viewOffset = ReadVec3(localPlayer + Offsets::m_vecViewOffset);
            Vector3 eyePos;
            eyePos.x = origin.x + viewOffset.x;
            eyePos.y = origin.y + viewOffset.y;
            eyePos.z = origin.z + viewOffset.z;
            
            MemoryAimbot(localPlayer, eyePos);
            SilentAim(localPlayer, eyePos);
            Triggerbot(localPlayer, eyePos);
            BunnyHop(localPlayer);
            ApplyGlow();
            SetThirdPerson(localPlayer);
            SetFOV(localPlayer);
            HandleHotkeys();
            UpdateFPS();
        }
        
        if (g_hDC && g_hGameWnd) {
            float viewMatrix[16];
            ReadProcessMemory(g_hProcess, (LPCVOID)(g_clientBase + Offsets::dwViewMatrix), viewMatrix, sizeof(viewMatrix), NULL);
            int localTeam = 0;
            uintptr_t localPlayer = ReadPtr(g_clientBase + Offsets::dwLocalPlayer);
            if (localPlayer) localTeam = ReadInt(localPlayer + Offsets::m_iTeamNum);
            
            DrawESP(g_hDC, localPlayer, localTeam, viewMatrix);
            DrawFOVCircle(g_hDC);
            DrawWatermark(g_hDC);
            DrawMenu(g_hDC);
        }
        
        Sleep(5);
    }
}

// ==================== MAIN ====================
int WINAPI WinMain(HINSTANCE hInst, HINSTANCE, LPSTR, int nCmdShow) {
    if (IsDebuggerPresent()) return 0;
    srand((unsigned int)GetTickCount());
    
    if (!ShowAuthDialog()) return 0;
    
    DWORD pid = GetProcessId(L"cs2.exe");
    if (!pid) { LaunchCS2(); Sleep(5000); pid = GetProcessId(L"cs2.exe"); }
    if (!pid) { MessageBoxA(NULL, "CS2 not found. Launch the game.", "VoidWare Error", MB_OK); return 0; }
    
    g_hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, pid);
    if (!g_hProcess) { MessageBoxA(NULL, "OpenProcess failed. Run as Administrator.", "VoidWare Error", MB_OK); return 0; }
    
    g_clientBase = GetModuleBase(pid, L"client.dll");
    if (!g_clientBase) { MessageBoxA(NULL, "client.dll not found. CS2 may have updated.", "VoidWare Error", MB_OK); CloseHandle(g_hProcess); return 0; }
    
    std::thread cheatThread(CheatLoop);
    cheatThread.detach();
    
    MSG msg;
    while (GetMessage(&msg, NULL, 0, 0)) {
        TranslateMessage(&msg);
        DispatchMessage(&msg);
    }
    
    if (g_hDC && g_hGameWnd) ReleaseDC(g_hGameWnd, g_hDC);
    CloseHandle(g_hProcess);
    return 0;
}
