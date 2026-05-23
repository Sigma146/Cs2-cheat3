// ============================================================
// VOIDWARE - CS2 UNDETECTABLE CHEAT (FULLY FIXED)
// Fixed: Vector3 initialization, bone matrix offsets, type conversions
// Compile: cl /EHsc /std:c++17 /MT /O2 /GL voidware_final.cpp user32.lib gdi32.lib winhttp.lib winmm.lib shell32.lib
// ============================================================

#include <Windows.h>
#include <TlHelp32.h>
#include <string>
#include <vector>
#include <cmath>
#include <fstream>
#include <sstream>
#include <thread>
#include <chrono>
#include <random>
#include <intrin.h>
#include <winhttp.h>
#include <shellapi.h>
#include <mutex>

#pragma comment(lib, "winmm.lib")
#pragma comment(lib, "winhttp.lib")
#pragma comment(lib, "shell32.lib")

// ==================== VOIDWARE BRANDING ====================
#define CHEAT_NAME "VoidWare"
#define CHEAT_VERSION "1.0"
#define CHEAT_COLOR_R 156
#define CHEAT_COLOR_G 39
#define CHEAT_COLOR_B 176

// ==================== KEYAUTH CONFIGURATION ====================
#define KEYAUTH_NAME "VoidWare"
#define KEYAUTH_OWNERID "XfwwmtO8U3"
#define KEYAUTH_SECRET "45d0249b058e9d0734c00e6f23b2f7c518e4322a658d222166d4e244f4887d85"
#define KEYAUTH_VERSION "1.0"

#define KEYAUTH_API "keyauth.win"
#define KEYAUTH_INIT "/api/1.2/init.php"
#define KEYAUTH_LICENSE "/api/1.2/license.php"

// ==================== ANTI-DETECTION ====================
#define RANDOM_DELAY(min, max) Sleep(min + (rand() % (max - min)))

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

// ==================== AIM POINT ENUM ====================
enum AimPoint { AIM_HEAD = 0, AIM_NECK = 1, AIM_TORSO = 2, AIM_PELVIS = 3 };
const char* AimPointNames[] = { "Head", "Neck", "Torso", "Pelvis" };
const int AimPointBones[] = { 6, 7, 5, 0 };

// ==================== FEATURE TOGGLES ====================
struct AimSettings {
    bool silentAim = false;
    bool memoryAimbot = true;
    int smoothness = 5;
    int fov = 30;
    int aimKey = VK_XBUTTON1;
    bool fovCircle = true;
    int fovCircleRadius = 30;
    AimPoint aimPoint = AIM_HEAD;
} g_aim;

struct VisualSettings {
    bool espBox = true;
    bool espHealth = true;
    bool espName = true;
    bool espDistance = true;
    bool teamCheck = true;
    bool glow = true;
    bool thirdPerson = false;
    float thirdPersonDistance = 100.0f;
    int gameFOV = 90;
} g_visual;

struct RageSettings {
    bool bhop = true;
    bool triggerbot = false;
    int triggerKey = VK_XBUTTON2;
    int triggerDelay = 20;
} g_rage;

struct WatermarkInfo {
    int fps = 0;
    int cheatFps = 0;
} g_watermark;

// ==================== GLOBALS ====================
HANDLE g_hProcess = NULL;
uintptr_t g_clientBase = 0;
HWND g_hGameWnd = NULL;
HDC g_hDC = NULL;
int g_screenWidth = 1920;
int g_screenHeight = 1080;
bool g_menuOpen = true;
int g_currentTab = 0;
std::mutex g_mutex;

// ==================== VECTOR3 STRUCTURE WITH CONSTRUCTOR ====================
struct Vector3 {
    float x, y, z;
    
    // Constructors
    Vector3() : x(0), y(0), z(0) {}
    Vector3(float x, float y, float z) : x(x), y(y), z(z) {}
    
    // Arithmetic operators
    Vector3 operator+(const Vector3& v) const { return Vector3(x + v.x, y + v.y, z + v.z); }
    Vector3 operator-(const Vector3& v) const { return Vector3(x - v.x, y - v.y, z - v.z); }
    Vector3 operator*(float s) const { return Vector3(x * s, y * s, z * s); }
    Vector3 operator/(float s) const { 
        if (s == 0) return Vector3(0, 0, 0);
        return Vector3(x / s, y / s, z / s); 
    }
    
    // Assignment operators
    Vector3& operator+=(const Vector3& v) { x += v.x; y += v.y; z += v.z; return *this; }
    Vector3& operator-=(const Vector3& v) { x -= v.x; y -= v.y; z -= v.z; return *this; }
    Vector3& operator*=(float s) { x *= s; y *= s; z *= s; return *this; }
    Vector3& operator/=(float s) { if (s != 0) { x /= s; y /= s; z /= s; } return *this; }
    
    // Utility functions
    float Length() const { return sqrtf(x*x + y*y + z*z); }
    void Normalize() { float l = Length(); if (l > 0.001f) { x /= l; y /= l; z /= l; } }
    float DistTo(const Vector3& v) const { 
        float dx = x - v.x, dy = y - v.y, dz = z - v.z; 
        return sqrtf(dx*dx + dy*dy + dz*dz); 
    }
};

// ==================== HWID GENERATION ====================
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

// ==================== KEYAUTH HTTP REQUESTS ====================
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
    WinHttpSendRequest(hRequest, headers, wcslen(headers), (LPVOID)data.c_str(), (DWORD)data.length(), (DWORD)data.length(), 0);
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
    return (response.find("\"success\":true") != std::string::npos);
}

bool KeyAuthLicense(const std::string& licenseKey, const std::string& hwid) {
    std::string postData = "type=license&key=" + licenseKey + 
                           "&hwid=" + hwid + 
                           "&name=" + std::string(KEYAUTH_NAME) + 
                           "&ownerid=" + std::string(KEYAUTH_OWNERID) + 
                           "&ver=" + std::string(KEYAUTH_VERSION);
    std::string response = HTTPPost(KEYAUTH_API, KEYAUTH_LICENSE, postData);
    return (response.find("\"success\":true") != std::string::npos);
}

bool ShowAuthDialog() {
    AllocConsole();
    FILE* f;
    freopen_s(&f, "CONOUT$", "w", stdout);
    SetConsoleTitle(L"VoidWare - Authentication");
    
    printf("\n");
    printf("  ╔════════════════════════════════════════════════════════════╗\n");
    printf("  ║                      VOIDWARE v%s                         ║\n", CHEAT_VERSION);
    printf("  ║                  CS2 Undetectable Cheat                    ║\n");
    printf("  ╚════════════════════════════════════════════════════════════╝\n\n");
    
    std::string hwid = GenerateHWID();
    printf("  HWID: %s\n\n", hwid.c_str());
    printf("  Enter License Key: ");
    
    char key[256];
    scanf_s("%s", key, (unsigned int)sizeof(key));
    printf("\n  [*] Verifying license key...\n");
    
    if (!KeyAuthInit()) { 
        printf("  [ERROR] Failed to connect to auth server\n"); 
        Sleep(2000); 
        fclose(f);
        FreeConsole();
        return false; 
    }
    
    if (KeyAuthLicense(key, hwid)) {
        printf("  [SUCCESS] License key verified!\n");
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
    Vector3 delta = dst - src;
    Vector3 angles;
    angles.y = atan2f(delta.y, delta.x) * 57.295779513f;
    angles.x = -atan2f(delta.z, sqrtf(delta.x*delta.x + delta.y*delta.y)) * 57.295779513f;
    angles.z = 0;
    
    if (angles.x > 89.0f) angles.x = 89.0f;
    if (angles.x < -89.0f) angles.x = -89.0f;
    while (angles.y > 180.0f) angles.y -= 360.0f;
    while (angles.y < -180.0f) angles.y += 360.0f;
    return angles;
}

float GetFov(Vector3 viewAngle, Vector3 aimAngle) {
    Vector3 delta = aimAngle - viewAngle;
    return sqrtf(delta.x*delta.x + delta.y*delta.y);
}

// ==================== MEMORY FUNCTIONS ====================
template<typename T>
T Read(uintptr_t address) {
    T buffer = {0};
    if (g_hProcess && address) {
        ReadProcessMemory(g_hProcess, (LPCVOID)address, &buffer, sizeof(T), NULL);
    }
    return buffer;
}

template<typename T>
void Write(uintptr_t address, T value) {
    if (g_hProcess && address) {
        WriteProcessMemory(g_hProcess, (LPVOID)address, &value, sizeof(T), NULL);
    }
}

// FIXED: Correct bone matrix offsets
Vector3 GetBonePosition(uintptr_t entity, int boneId) {
    uintptr_t boneMatrix = Read<uintptr_t>(entity + 0x280);
    if (!boneMatrix) return Vector3();
    
    // Correct offsets for bone matrix (CS2 uses 0x30 stride)
    float x = Read<float>(boneMatrix + boneId * 0x30 + 0x0C);
    float y = Read<float>(boneMatrix + boneId * 0x30 + 0x1C);
    float z = Read<float>(boneMatrix + boneId * 0x30 + 0x2C);
    
    return Vector3(x, y, z);
}

Vector3 GetAimPoint(uintptr_t entity, AimPoint aimPoint) {
    int boneId = AimPointBones[(int)aimPoint];
    return GetBonePosition(entity, boneId);
}

// ==================== AIMBOT ====================
uintptr_t GetBestTarget(uintptr_t localPlayer, Vector3 localEyePos, float maxFov, AimPoint aimPoint, Vector3& outAimPos) {
    uintptr_t bestEntity = 0;
    float bestFov = maxFov;
    Vector3 localAngles = Read<Vector3>(localPlayer + Offsets::m_angEyeAngles);
    
    for (int i = 1; i <= 64; i++) {
        uintptr_t entity = Read<uintptr_t>(g_clientBase + Offsets::dwEntityList + i * 0x8);
        if (!entity) continue;
        
        int health = Read<int>(entity + Offsets::m_iHealth);
        if (health <= 0 || health > 100) continue;
        
        int team = Read<int>(entity + Offsets::m_iTeamNum);
        int localTeam = Read<int>(localPlayer + Offsets::m_iTeamNum);
        if (team == localTeam) continue;
        
        bool dormant = Read<bool>(entity + Offsets::m_bDormant);
        if (dormant) continue;
        
        Vector3 aimPos = GetAimPoint(entity, aimPoint);
        if (aimPos.x == 0 && aimPos.y == 0 && aimPos.z == 0) {
            Vector3 origin = Read<Vector3>(entity + Offsets::m_vecOrigin);
            Vector3 viewOffset = Read<Vector3>(entity + Offsets::m_vecViewOffset);
            aimPos = origin + viewOffset;
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
    if (!g_aim.memoryAimbot) return;
    if (!(GetAsyncKeyState(g_aim.aimKey) & 0x8000)) return;
    
    Vector3 aimPos;
    uintptr_t target = GetBestTarget(localPlayer, localEyePos, (float)g_aim.fov, g_aim.aimPoint, aimPos);
    if (!target) return;
    
    Vector3 aimAngle = CalcAngle(localEyePos, aimPos);
    Vector3 currentAngle = Read<Vector3>(localPlayer + Offsets::m_angEyeAngles);
    
    if (g_aim.smoothness > 1) {
        Vector3 delta = aimAngle - currentAngle;
        float len = delta.Length();
        if (len > 0.001f) {
            delta = delta / len;
            aimAngle = currentAngle + delta / (float)g_aim.smoothness;
        }
    }
    
    Write<Vector3>(localPlayer + Offsets::m_angEyeAngles, aimAngle);
}

void SilentAim(uintptr_t localPlayer, Vector3 localEyePos) {
    if (!g_aim.silentAim) return;
    if (!(GetAsyncKeyState(g_aim.aimKey) & 0x8000)) return;
    
    Vector3 aimPos;
    uintptr_t target = GetBestTarget(localPlayer, localEyePos, (float)g_aim.fov, g_aim.aimPoint, aimPos);
    if (!target) return;
    
    Vector3 aimAngle = CalcAngle(localEyePos, aimPos);
    uintptr_t clientState = Read<uintptr_t>(g_clientBase + Offsets::dwClientState);
    if (clientState) {
        Vector3 punch = Read<Vector3>(localPlayer + Offsets::m_aimPunchAngle);
        aimAngle = aimAngle - (punch * 2.0f);
        Write<Vector3>(clientState + 0x4D88, aimAngle);
    }
}

// ==================== TRIGGERBOT ====================
bool IsCrosshairOnEnemy(uintptr_t localPlayer, Vector3 localEyePos) {
    Vector3 aimPos;
    uintptr_t target = GetBestTarget(localPlayer, localEyePos, 2.0f, g_aim.aimPoint, aimPos);
    if (!target) return false;
    
    Vector3 localAngles = Read<Vector3>(localPlayer + Offsets::m_angEyeAngles);
    Vector3 aimAngle = CalcAngle(localEyePos, aimPos);
    return GetFov(localAngles, aimAngle) < 2.0f;
}

void Triggerbot(uintptr_t localPlayer, Vector3 localEyePos) {
    if (!g_rage.triggerbot) return;
    if (!(GetAsyncKeyState(g_rage.triggerKey) & 0x8000)) return;
    if (!IsCrosshairOnEnemy(localPlayer, localEyePos)) return;
    
    uintptr_t activeWeapon = Read<uintptr_t>(localPlayer + Offsets::m_hActiveWeapon);
    if (!activeWeapon) return;
    
    float nextAttack = Read<float>(activeWeapon + Offsets::m_flNextPrimaryAttack);
    float curTime = Read<float>(g_clientBase + Offsets::dwGlobalVars + 0x8);
    if (nextAttack > curTime) return;
    
    int clip = Read<int>(activeWeapon + Offsets::m_iClip1);
    if (clip <= 0) return;
    
    static DWORD lastShot = 0;
    DWORD now = GetTickCount();
    if (now - lastShot >= (DWORD)g_rage.triggerDelay) {
        Write<uintptr_t>(g_clientBase + Offsets::dwForceAttack, 6);
        Sleep(1);
        Write<uintptr_t>(g_clientBase + Offsets::dwForceAttack, 4);
        lastShot = now;
    }
}

// ==================== BUNNY HOP ====================
void BunnyHop(uintptr_t localPlayer) {
    if (!g_rage.bhop) return;
    if (!(GetAsyncKeyState(VK_SPACE) & 0x8000)) return;
    
    int flags = Read<int>(localPlayer + Offsets::m_fFlags);
    if (flags & (1 << 0)) {
        Write<uintptr_t>(g_clientBase + Offsets::dwForceJump, 6);
        Sleep(1);
        Write<uintptr_t>(g_clientBase + Offsets::dwForceJump, 4);
    }
}

// ==================== 3RD PERSON & FOV ====================
void SetThirdPerson(uintptr_t localPlayer) {
    if (!g_visual.thirdPerson) {
        uintptr_t cameraServices = Read<uintptr_t>(localPlayer + 0x38);
        if (cameraServices) {
            Write<int>(cameraServices + 0x10, 0);
        }
        return;
    }
    
    uintptr_t cameraServices = Read<uintptr_t>(localPlayer + 0x38);
    if (cameraServices) {
        Write<int>(cameraServices + 0x10, 1);
        Write<float>(cameraServices + 0x14, g_visual.thirdPersonDistance);
    }
}

void SetFOV(uintptr_t localPlayer) {
    uintptr_t cameraServices = Read<uintptr_t>(localPlayer + 0x38);
    if (cameraServices) {
        float newFOV = (float)g_visual.gameFOV;
        if (newFOV < 70) newFOV = 70;
        if (newFOV > 179) newFOV = 179;
        Write<float>(cameraServices + 0x18, newFOV);
        Write<float>(cameraServices + 0x1C, newFOV);
    }
}

// ==================== GLOW ESP ====================
void ApplyGlow() {
    if (!g_visual.glow) return;
    
    uintptr_t localPlayer = Read<uintptr_t>(g_clientBase + Offsets::dwLocalPlayer);
    if (!localPlayer) return;
    
    int localTeam = Read<int>(localPlayer + Offsets::m_iTeamNum);
    uintptr_t glowManager = Read<uintptr_t>(g_clientBase + 0x18D6048);
    if (!glowManager) return;
    
    for (int i = 0; i < 64; i++) {
        uintptr_t entity = Read<uintptr_t>(g_clientBase + Offsets::dwEntityList + i * 0x8);
        if (!entity) continue;
        
        int health = Read<int>(entity + Offsets::m_iHealth);
        if (health <= 0 || health > 100) continue;
        
        int team = Read<int>(entity + Offsets::m_iTeamNum);
        if (g_visual.teamCheck && team == localTeam) continue;
        
        int glowIndex = Read<int>(entity + 0x328);
        if (glowIndex == -1) continue;
        
        uintptr_t glowObject = glowManager + (glowIndex * 0x38);
        bool isEnemy = (team != localTeam);
        
        Write<float>(glowObject + 0x8, isEnemy ? 1.0f : 0.0f);
        Write<float>(glowObject + 0xC, isEnemy ? 0.0f : 1.0f);
        Write<float>(glowObject + 0x10, 0.0f);
        Write<float>(glowObject + 0x14, 0.8f);
        Write<bool>(glowObject + 0x28, true);
        Write<bool>(glowObject + 0x29, false);
    }
}

// ==================== WATERMARK ====================
void UpdateWatermark() {
    static DWORD lastFPSTime = GetTickCount();
    static int fpsCounter = 0;
    
    fpsCounter++;
    DWORD now = GetTickCount();
    if (now - lastFPSTime >= 1000) {
        g_watermark.fps = fpsCounter;
        fpsCounter = 0;
        lastFPSTime = now;
    }
}

// ==================== DRAWING FUNCTIONS ====================
void DrawLine(HDC hdc, int x1, int y1, int x2, int y2, COLORREF color) {
    HPEN pen = CreatePen(PS_SOLID, 1, color);
    HPEN oldPen = (HPEN)SelectObject(hdc, pen);
    MoveToEx(hdc, x1, y1, NULL);
    LineTo(hdc, x2, y2);
    SelectObject(hdc, oldPen);
    DeleteObject(pen);
}

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

void DrawFOVCircle(HDC hdc) {
    if (!g_aim.fovCircle) return;
    if (!g_hGameWnd) return;
    
    POINT cursor;
    GetCursorPos(&cursor);
    ScreenToClient(g_hGameWnd, &cursor);
    
    HPEN pen = CreatePen(PS_SOLID, 1, RGB(CHEAT_COLOR_R, CHEAT_COLOR_G, CHEAT_COLOR_B));
    HPEN oldPen = (HPEN)SelectObject(hdc, pen);
    HBRUSH oldBrush = (HBRUSH)SelectObject(hdc, GetStockObject(NULL_BRUSH));
    
    Ellipse(hdc, cursor.x - g_aim.fovCircleRadius, cursor.y - g_aim.fovCircleRadius,
                 cursor.x + g_aim.fovCircleRadius, cursor.y + g_aim.fovCircleRadius);
    
    SelectObject(hdc, oldPen);
    SelectObject(hdc, oldBrush);
    DeleteObject(pen);
}

void DrawWatermark(HDC hdc) {
    char watermark[256];
    sprintf_s(watermark, sizeof(watermark), "VoidWare v%s | FPS: %d | Aim: %s | 3rd: %s | FOV: %d",
        CHEAT_VERSION, g_watermark.fps, AimPointNames[(int)g_aim.aimPoint],
        g_visual.thirdPerson ? "ON" : "OFF", g_visual.gameFOV);
    
    SetTextColor(hdc, RGB(0, 0, 0));
    SetBkMode(hdc, TRANSPARENT);
    TextOutA(hdc, 11, 11, watermark, (int)strlen(watermark));
    SetTextColor(hdc, RGB(CHEAT_COLOR_R, CHEAT_COLOR_G, CHEAT_COLOR_B));
    TextOutA(hdc, 10, 10, watermark, (int)strlen(watermark));
}

bool WorldToScreen(const Vector3& pos, Vector3& screen, float* viewMatrix) {
    if (!viewMatrix) return false;
    
    float w = viewMatrix[12] * pos.x + viewMatrix[13] * pos.y + viewMatrix[14] * pos.z + viewMatrix[15];
    if (w < 0.01f) return false;
    
    float invW = 1.0f / w;
    screen.x = (float)(g_screenWidth / 2) + (0.5f * (viewMatrix[0] * pos.x + viewMatrix[1] * pos.y + viewMatrix[2] * pos.z + viewMatrix[3]) * invW * (float)g_screenWidth);
    screen.y = (float)(g_screenHeight / 2) - (0.5f * (viewMatrix[4] * pos.x + viewMatrix[5] * pos.y + viewMatrix[6] * pos.z + viewMatrix[7]) * invW * (float)g_screenHeight);
    return true;
}

void DrawESP(HDC hdc, uintptr_t localPlayer, int localTeam, float* viewMatrix) {
    if (!viewMatrix) return;
    
    for (int i = 1; i <= 64; i++) {
        uintptr_t entity = Read<uintptr_t>(g_clientBase + Offsets::dwEntityList + i * 0x8);
        if (!entity) continue;
        
        int health = Read<int>(entity + Offsets::m_iHealth);
        if (health <= 0 || health > 100) continue;
        
        int team = Read<int>(entity + Offsets::m_iTeamNum);
        if (g_visual.teamCheck && team == localTeam) continue;
        
        Vector3 origin = Read<Vector3>(entity + Offsets::m_vecOrigin);
        Vector3 headPos = origin + Read<Vector3>(entity + Offsets::m_vecViewOffset);
        
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
        
        bool isEnemy = (team != localTeam);
        COLORREF color = isEnemy ? RGB(255, 0, 0) : RGB(0, 255, 0);
        
        if (g_visual.espBox) {
            DrawRect(hdc, x, y, w, h, color);
        }
        
        if (g_visual.espHealth) {
            int healthHeight = (int)(h * (health / 100.0f));
            if (healthHeight > 0) {
                DrawFilledRect(hdc, x - 6, y + h - healthHeight, 4, healthHeight, RGB(0, 255, 0));
            }
        }
        
        if (g_visual.espDistance) {
            char dist[32];
            float distance = origin.DistTo(Read<Vector3>(localPlayer + Offsets::m_vecOrigin));
            sprintf_s(dist, sizeof(dist), "%.0fm", distance);
            DrawText(hdc, x + w / 2 - 15, y + h + 2, dist, RGB(255, 255, 255));
        }
    }
}

// ==================== IN-GAME MENU ====================
void DrawMenu(HDC hdc) {
    if (!g_menuOpen) return;
    
    DrawFilledRect(hdc, 50, 50, 350, 400, RGB(30, 30, 40));
    DrawRect(hdc, 50, 50, 350, 400, RGB(CHEAT_COLOR_R, CHEAT_COLOR_G, CHEAT_COLOR_B));
    
    char title[64];
    sprintf_s(title, sizeof(title), "VOIDWARE v%s - INS TO TOGGLE", CHEAT_VERSION);
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
        DrawText(hdc, 70, y + 25, ("Current: " + std::string(AimPointNames[(int)g_aim.aimPoint])).c_str(), RGB(CHEAT_COLOR_R, CHEAT_COLOR_G, CHEAT_COLOR_B));
        DrawText(hdc, 70, y + 55, ("[F2] Aim FOV: " + std::to_string(g_aim.fov)).c_str(), RGB(200, 200, 200));
        DrawText(hdc, 70, y + 80, ("[F3] Smoothness: " + std::to_string(g_aim.smoothness)).c_str(), RGB(200, 200, 200));
        DrawText(hdc, 70, y + 105, "[F4] Toggle Silent Aim", RGB(200, 200, 200));
        DrawText(hdc, 200, y + 105, g_aim.silentAim ? "[ON]" : "[OFF]", g_aim.silentAim ? RGB(0,255,0) : RGB(255,0,0));
        DrawText(hdc, 70, y + 130, "[F5] Toggle FOV Circle", RGB(200, 200, 200));
        DrawText(hdc, 200, y + 130, g_aim.fovCircle ? "[ON]" : "[OFF]", g_aim.fovCircle ? RGB(0,255,0) : RGB(255,0,0));
        DrawText(hdc, 70, y + 155, "[F6] Toggle Memory Aim", RGB(200, 200, 200));
        DrawText(hdc, 200, y + 155, g_aim.memoryAimbot ? "[ON]" : "[OFF]", g_aim.memoryAimbot ? RGB(0,255,0) : RGB(255,0,0));
    }
    else if (g_currentTab == 1) {
        DrawText(hdc, 70, y, "[F1] Toggle ESP Box", RGB(200, 200, 200));
        DrawText(hdc, 200, y, g_visual.espBox ? "[ON]" : "[OFF]", g_visual.espBox ? RGB(0,255,0) : RGB(255,0,0));
        DrawText(hdc, 70, y + 25, "[F2] Toggle ESP Health", RGB(200, 200, 200));
        DrawText(hdc, 200, y + 25, g_visual.espHealth ? "[ON]" : "[OFF]", g_visual.espHealth ? RGB(0,255,0) : RGB(255,0,0));
        DrawText(hdc, 70, y + 50, "[F3] Toggle ESP Distance", RGB(200, 200, 200));
        DrawText(hdc, 200, y + 50, g_visual.espDistance ? "[ON]" : "[OFF]", g_visual.espDistance ? RGB(0,255,0) : RGB(255,0,0));
        DrawText(hdc, 70, y + 75, "[F4] Toggle Glow ESP", RGB(200, 200, 200));
        DrawText(hdc, 200, y + 75, g_visual.glow ? "[ON]" : "[OFF]", g_visual.glow ? RGB(0,255,0) : RGB(255,0,0));
        DrawText(hdc, 70, y + 100, "[F5] Toggle 3rd Person", RGB(200, 200, 200));
        DrawText(hdc, 200, y + 100, g_visual.thirdPerson ? "[ON]" : "[OFF]", g_visual.thirdPerson ? RGB(0,255,0) : RGB(255,0,0));
        DrawText(hdc, 70, y + 125, "[F6] Toggle Team Check", RGB(200, 200, 200));
        DrawText(hdc, 200, y + 125, g_visual.teamCheck ? "[ON]" : "[OFF]", g_visual.teamCheck ? RGB(0,255,0) : RGB(255,0,0));
    }
    else if (g_currentTab == 2) {
        DrawText(hdc, 70, y, "[F1] Toggle Bunny Hop", RGB(200, 200, 200));
        DrawText(hdc, 200, y, g_rage.bhop ? "[ON]" : "[OFF]", g_rage.bhop ? RGB(0,255,0) : RGB(255,0,0));
        DrawText(hdc, 70, y + 25, "[F2] Toggle Triggerbot", RGB(200, 200, 200));
        DrawText(hdc, 200, y + 25, g_rage.triggerbot ? "[ON]" : "[OFF]", g_rage.triggerbot ? RGB(0,255,0) : RGB(255,0,0));
        DrawText(hdc, 70, y + 55, ("[F3] Trigger Delay: " + std::to_string(g_rage.triggerDelay) + "ms").c_str(), RGB(200, 200, 200));
    }
    
    DrawText(hdc, 70, 380, "[F12] Save Config", RGB(200, 200, 200));
    DrawText(hdc, 70, 395, "[END] Exit", RGB(200, 200, 200));
}

// ==================== HOTKEY HANDLER ====================
void HandleHotkeys() {
    if (GetAsyncKeyState(VK_F1) & 1) {
        std::lock_guard<std::mutex> lock(g_mutex);
        if (g_currentTab == 0) {
            g_aim.aimPoint = (AimPoint)(((int)g_aim.aimPoint + 1) % 4);
        } else if (g_currentTab == 1) {
            g_visual.espBox = !g_visual.espBox;
        } else if (g_currentTab == 2) {
            g_rage.bhop = !g_rage.bhop;
        }
    }
    if (GetAsyncKeyState(VK_F2) & 1) {
        std::lock_guard<std::mutex> lock(g_mutex);
        if (g_currentTab == 0) {
            g_aim.fov += 5; if (g_aim.fov > 180) g_aim.fov = 5;
        } else if (g_currentTab == 1) {
            g_visual.espHealth = !g_visual.espHealth;
        } else if (g_currentTab == 2) {
            g_rage.triggerbot = !g_rage.triggerbot;
        }
    }
    if (GetAsyncKeyState(VK_F3) & 1) {
        std::lock_guard<std::mutex> lock(g_mutex);
        if (g_currentTab == 0) {
            g_aim.smoothness += 1; if (g_aim.smoothness > 20) g_aim.smoothness = 1;
        } else if (g_currentTab == 1) {
            g_visual.espDistance = !g_visual.espDistance;
        } else if (g_currentTab == 2) {
            g_rage.triggerDelay += 10; if (g_rage.triggerDelay > 200) g_rage.triggerDelay = 10;
        }
    }
    if (GetAsyncKeyState(VK_F4) & 1) {
        std::lock_guard<std::mutex> lock(g_mutex);
        if (g_currentTab == 0) g_aim.silentAim = !g_aim.silentAim;
        else if (g_currentTab == 1) g_visual.glow = !g_visual.glow;
    }
    if (GetAsyncKeyState(VK_F5) & 1) {
        std::lock_guard<std::mutex> lock(g_mutex);
        if (g_currentTab == 0) g_aim.fovCircle = !g_aim.fovCircle;
        else if (g_currentTab == 1) g_visual.thirdPerson = !g_visual.thirdPerson;
    }
    if (GetAsyncKeyState(VK_F6) & 1) {
        std::lock_guard<std::mutex> lock(g_mutex);
        if (g_currentTab == 0) g_aim.memoryAimbot = !g_aim.memoryAimbot;
        else if (g_currentTab == 1) g_visual.teamCheck = !g_visual.teamCheck;
    }
    if (GetAsyncKeyState(VK_INSERT) & 1) {
        g_menuOpen = !g_menuOpen;
    }
    if (GetAsyncKeyState(VK_END) & 1) {
        exit(0);
    }
    
    // FOV adjustment keys
    if (GetAsyncKeyState(VK_PRIOR) & 1) { 
        std::lock_guard<std::mutex> lock(g_mutex);
        g_visual.gameFOV += 5; 
        if (g_visual.gameFOV > 179) g_visual.gameFOV = 179; 
    }
    if (GetAsyncKeyState(VK_NEXT) & 1) { 
        std::lock_guard<std::mutex> lock(g_mutex);
        g_visual.gameFOV -= 5; 
        if (g_visual.gameFOV < 70) g_visual.gameFOV = 70; 
    }
    
    // 3rd person distance adjustment
    if ((GetAsyncKeyState(VK_ADD) & 1) || (GetAsyncKeyState(VK_OEM_PLUS) & 1)) { 
        std::lock_guard<std::mutex> lock(g_mutex);
        g_visual.thirdPersonDistance += 10; 
        if (g_visual.thirdPersonDistance > 300) g_visual.thirdPersonDistance = 300; 
    }
    if ((GetAsyncKeyState(VK_SUBTRACT) & 1) || (GetAsyncKeyState(VK_OEM_MINUS) & 1)) { 
        std::lock_guard<std::mutex> lock(g_mutex);
        g_visual.thirdPersonDistance -= 10; 
        if (g_visual.thirdPersonDistance < 30) g_visual.thirdPersonDistance = 30; 
    }
    
    // Tab switching
    if (GetAsyncKeyState('1') & 1) g_currentTab = 0;
    if (GetAsyncKeyState('2') & 1) g_currentTab = 1;
    if (GetAsyncKeyState('3') & 1) g_currentTab = 2;
}

// ==================== HIDE FROM ANTI-CHEAT ====================
void HideThread() {
    HMODULE ntdll = GetModuleHandleA("ntdll.dll");
    if (ntdll) {
        auto NtSetInformationThread = (NTSTATUS(NTAPI*)(HANDLE, ULONG, PVOID, ULONG))GetProcAddress(ntdll, "NtSetInformationThread");
        if (NtSetInformationThread) {
            NtSetInformationThread(GetCurrentThread(), 0x11, NULL, 0);
        }
    }
}

void PatchETW() {
    HMODULE ntdll = GetModuleHandleA("ntdll.dll");
    if (ntdll) {
        BYTE* etwEventWrite = (BYTE*)GetProcAddress(ntdll, "EtwEventWrite");
        if (etwEventWrite) {
            DWORD oldProtect;
            VirtualProtect(etwEventWrite, 5, PAGE_EXECUTE_READWRITE, &oldProtect);
            etwEventWrite[0] = 0x31;
            etwEventWrite[1] = 0xC0;
            etwEventWrite[2] = 0xC3;
            VirtualProtect(etwEventWrite, 5, oldProtect, &oldProtect);
        }
    }
}

// ==================== PROCESS FUNCTIONS ====================
DWORD GetProcessId(const wchar_t* name) {
    PROCESSENTRY32W pe = { sizeof(pe) };
    HANDLE snap = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
    if (snap == INVALID_HANDLE_VALUE) return 0;
    
    DWORD pid = 0;
    if (Process32FirstW(snap, &pe)) {
        do {
            if (_wcsicmp(pe.szExeFile, name) == 0) {
                pid = pe.th32ProcessID;
                break;
            }
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
            if (_wcsicmp(me.szModule, moduleName) == 0) {
                base = (uintptr_t)me.modBaseAddr;
                break;
            }
        } while (Module32NextW(snap, &me));
    }
    CloseHandle(snap);
    return base;
}

void LaunchCS2() {
    ShellExecuteW(NULL, L"open", L"steam://rungameid/730", NULL, NULL, SW_SHOW);
}

// ==================== MAIN CHEAT LOOP ====================
void CheatLoop() {
    HideThread();
    PatchETW();
    
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
        
        uintptr_t localPlayer = Read<uintptr_t>(g_clientBase + Offsets::dwLocalPlayer);
        if (localPlayer) {
            Vector3 origin = Read<Vector3>(localPlayer + Offsets::m_vecOrigin);
            Vector3 viewOffset = Read<Vector3>(localPlayer + Offsets::m_vecViewOffset);
            Vector3 eyePos = origin + viewOffset;
            
            MemoryAimbot(localPlayer, eyePos);
            SilentAim(localPlayer, eyePos);
            Triggerbot(localPlayer, eyePos);
            BunnyHop(localPlayer);
            ApplyGlow();
            SetThirdPerson(localPlayer);
            SetFOV(localPlayer);
            HandleHotkeys();
            UpdateWatermark();
        }
        
        if (g_hDC && g_hGameWnd) {
            float viewMatrix[16];
            ReadProcessMemory(g_hProcess, (LPCVOID)(g_clientBase + Offsets::dwViewMatrix), viewMatrix, sizeof(viewMatrix), NULL);
            
            int localTeam = 0;
            uintptr_t localPlayer = Read<uintptr_t>(g_clientBase + Offsets::dwLocalPlayer);
            if (localPlayer) localTeam = Read<int>(localPlayer + Offsets::m_iTeamNum);
            
            DrawESP(g_hDC, localPlayer, localTeam, viewMatrix);
            DrawFOVCircle(g_hDC);
            DrawWatermark(g_hDC);
            DrawMenu(g_hDC);
        }
        
        RANDOM_DELAY(1, 5);
    }
}

// ==================== MAIN ENTRY POINT ====================
int WINAPI WinMain(HINSTANCE hInst, HINSTANCE, LPSTR, int nCmdShow) {
    if (IsDebuggerPresent()) return 0;
    srand((unsigned int)GetTickCount());
    
    if (!ShowAuthDialog()) return 0;
    
    DWORD pid = GetProcessId(L"cs2.exe");
    if (!pid) { 
        LaunchCS2(); 
        Sleep(5000); 
        pid = GetProcessId(L"cs2.exe"); 
    }
    if (!pid) { 
        MessageBoxA(NULL, "Failed to find CS2 process. Make sure the game is running.", "VoidWare Error", MB_OK); 
        return 0; 
    }
    
    g_hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, pid);
    if (!g_hProcess) { 
        MessageBoxA(NULL, "Failed to open CS2 process. Run as Administrator.", "VoidWare Error", MB_OK); 
        return 0; 
    }
    
    g_clientBase = GetModuleBase(pid, L"client.dll");
    if (!g_clientBase) { 
        MessageBoxA(NULL, "Failed to get client.dll base address. CS2 may have updated.", "VoidWare Error", MB_OK); 
        CloseHandle(g_hProcess);
        return 0; 
    }
    
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

#pragma comment(linker, "/SUBSYSTEM:WINDOWS /ENTRY:WinMainCRTStartup")
