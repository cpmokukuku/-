// ==UserScript==
// @name         大连海事教务系统自动抢课脚本
// @namespace    http://tampermonkey.net/
// @version      2026-07-29
// @description  自动监测名额并抢课
// @author       You
// @match        *://*.dlmu.edu.cn/*
// @icon         data:image/gif;base64,R0lGODlhAQABAAAAACH5BAEKAAEALAAAAAABAAEAAAICTAEAOw==
// @grant        GM_setValue
// @grant        GM_getValue
// @run-at       document-start
// ==/UserScript==

(function() {
    'use strict';

    let TARGET_COURSE = GM_getValue("target_course", "");
    let isRunning = GM_getValue("is_running", false);
    const REFRESH_DELAY = 2500;
    let isClicked = false;
    let isSuccess = false;

    const autoConfirm = function(msg) {
        console.log("[自动确认]:", msg);
        return true;
    };

    const originalAlert = window.alert;
    const autoAlert = function(msg) {
        console.log("[拦截提示]:", msg);
        if (msg && (msg.includes("成功") || msg.includes("已选择") || msg.includes("冲突"))) {
            isSuccess = true;
            GM_setValue("is_running", false);
            updateUIState();
            console.log("选课交互完成！");
            originalAlert("选课交互完成！提示内容：" + msg);
        }
    };

    window.confirm = autoConfirm;
    window.alert = autoAlert;
    try {
        if (window.top && window.top !== window) {
            window.top.confirm = autoConfirm;
            window.top.alert = autoAlert;
        }
    } catch (e) {}

    function checkAndGrab() {
        if (!isRunning || isSuccess || isClicked || !TARGET_COURSE) {
            return;
        }

        const rows = document.querySelectorAll("tr.electGridTr");
        let found = false;

        rows.forEach(row => {
            const text = row.innerText || row.textContent;
            if (text.includes(TARGET_COURSE)) {
                found = true;

                if (text.includes("已选") || text.includes("退选")) {
                    console.log("课程已在列表中，停止抢课！");
                    isSuccess = true;
                    GM_setValue("is_running", false);
                    updateUIState();
                    return;
                }

                const countTd = row.querySelector("td.stdCount");
                if (countTd) {
                    const match = countTd.innerText.replace(/\s+/g, '').match(/(\d+)\/(\d+)/);
                    if (match) {
                        const current = parseInt(match[1], 10);
                        const max = parseInt(match[2], 10);
                        console.log(`[${TARGET_COURSE}] 人数: ${current}/${max}`);

                        if (current < max) {
                            console.log("发现名额，准备抢课！");
                            const allLinks = Array.from(row.querySelectorAll("a"));
                            const selectBtn = allLinks.find(a => a.innerText.trim().includes("选课"))
                                            || row.querySelector("a.lessonListOperator");

                            if (selectBtn) {
                                isClicked = true;
                                selectBtn.click();
                                return;
                            }
                        }
                    }
                }
            }
        });

        if (!found) {
            console.warn(`未找到课程“${TARGET_COURSE}”，请确认是否切换到对应选课分类。`);
        }

        if (isRunning && !isSuccess && !isClicked) {
            setTimeout(() => {
                window.location.reload();
            }, REFRESH_DELAY);
        }
    }

    function createUI() {
        if (document.getElementById("grab-course-panel")) return;

        const panel = document.createElement("div");
        panel.id = "grab-course-panel";
        panel.style.cssText = `
            position: fixed;
            top: 20px;
            right: 20px;
            z-index: 999999;
            background: #ffffff;
            border: 2px solid #005bac;
            border-radius: 8px;
            padding: 12px 16px;
            box-shadow: 0 4px 12px rgba(0,0,0,0.15);
            font-family: Arial, sans-serif;
            font-size: 14px;
            color: #333;
            width: 220px;
        `;

        panel.innerHTML = `
            <div style="font-weight: bold; margin-bottom: 8px; color: #005bac; display: flex; justify-content: space-between; align-items: center;">
                <span> 大连海事抢课助手</span>
                <span id="grab-status-tag" style="font-size: 12px; padding: 2px 6px; border-radius: 4px; background: #eee;">未运行</span>
            </div>
            <div style="margin-bottom: 8px;">
                <label style="display: block; font-size: 12px; margin-bottom: 4px; color: #666;">目标课程名称：</label>
                <input type="text" id="target-course-input" placeholder="请输入课程全称" value="${TARGET_COURSE}" style="width: 100%; box-sizing: border-box; padding: 5px; border: 1px solid #ccc; border-radius: 4px;" />
            </div>
            <button id="toggle-grab-btn" style="width: 100%; padding: 6px; border: none; border-radius: 4px; font-weight: bold; cursor: pointer; color: white; background: #28a745;">
                开始自动抢课
            </button>
        `;

        document.body.appendChild(panel);

        const input = document.getElementById("target-course-input");
        const btn = document.getElementById("toggle-grab-btn");

        input.addEventListener("input", (e) => {
            TARGET_COURSE = e.target.value.trim();
            GM_setValue("target_course", TARGET_COURSE);
        });

        btn.addEventListener("click", () => {
            if (!isRunning && !TARGET_COURSE) {
                originalAlert("⚠️ 请先在输入框中填写要抢的课程名称！");
                return;
            }

            isRunning = !isRunning;
            GM_setValue("is_running", isRunning);
            updateUIState();

            if (isRunning) {
                window.location.reload();
            }
        });

        updateUIState();
    }

    function updateUIState() {
        const tag = document.getElementById("grab-status-tag");
        const btn = document.getElementById("toggle-grab-btn");
        if (!tag || !btn) return;

        if (isRunning) {
            tag.innerText = "监控中...";
            tag.style.background = "#d4edda";
            tag.style.color = "#155724";
            btn.innerText = "停止抢课";
            btn.style.background = "#dc3545";
        } else {
            tag.innerText = "已暂停";
            tag.style.background = "#e2e3e5";
            tag.style.color = "#383d41";
            btn.innerText = "开始自动抢课";
            btn.style.background = "#28a745";
        }
    }

    function init() {
        createUI();
        if (isRunning && TARGET_COURSE) {
            setTimeout(checkAndGrab, 1000);
        }
    }

    if (document.readyState === "loading") {
        document.addEventListener("DOMContentLoaded", init);
    } else {
        init();
    }
})();