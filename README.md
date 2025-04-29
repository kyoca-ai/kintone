(function() {
    'use strict';

    kintone.events.on('space.portal.show', (event) => {
      const SPACE_ID = '7';
      const APP_ID_164 = 164;
      const APP_ID_127 = 127;

      if (event.spaceId !== SPACE_ID) {
        return event;
      }

      const el = kintone.space.portal.getContentSpaceElement();
      let selectedDate = luxon.DateTime.now().setZone('Asia/Tokyo');
      let currentRecords164 = [];

      const dateDisplay = document.createElement('div');
      dateDisplay.id = 'dateDisplay';

      const prevButton = document.createElement('span');
      prevButton.id = 'prevButton';
      prevButton.textContent = '◀';

      const dateText = document.createElement('span');
      dateText.id = 'dateText';

      const nextButton = document.createElement('span');
      nextButton.id = 'nextButton';
      nextButton.textContent = '▶';

      const organizationDisplay = document.createElement('span');
      organizationDisplay.id = 'organizationDisplay';

      dateDisplay.appendChild(prevButton);
      dateDisplay.appendChild(dateText);
      dateDisplay.appendChild(nextButton);
      dateDisplay.appendChild(organizationDisplay);
      el.appendChild(dateDisplay);

      function updateDateDisplay() {
        dateText.textContent = selectedDate.toFormat('yyyy年MM月dd日（EEE）');
      }

      dateText.addEventListener('click', () => {
        const targetDate = selectedDate.toFormat('yyyy-MM-dd');
        const query = `対象日 = "${targetDate}"`;
        const params = { app: APP_ID_164, query: query };

        kintone.api(kintone.api.url('/k/v1/records.json', true), 'GET', params)
          .then((resp) => {
            if (resp.records.length > 0) {
              const recordId = resp.records[0].$id.value;
              const recordUrl = `/k/${APP_ID_164}/show#record=${recordId}`;
              window.open(recordUrl, '_blank');
            } else {
              alert('該当するレコードがありません。');
            }
          })
          .catch((error) => {
            console.error(error);
            alert('レコード取得に失敗しました。');
          });
      });

      function render164Records(record) {
        const oldElements = el.querySelectorAll('.cardContainer, .noRecordMessage, .shampooSection, .raikouSection, .taikouSection');
        oldElements.forEach(elem => elem.remove());

        if (!record) {
          const noRecordMessage = document.createElement('div');
          noRecordMessage.textContent = '本日のデータはありません。';
          noRecordMessage.className = 'noRecordMessage';
          el.appendChild(noRecordMessage);
          return;
        }

        const fieldsToDisplay = ['_8_00_20_00', '宿直', '訓練犬', '宿泊犬', '保育'];
        const textFields = ['よのなかルール', 'メモ'];

        const cardContainer = document.createElement('div');
        cardContainer.className = 'cardContainer';

        fieldsToDisplay.forEach((fieldCode) => {
          const fieldValue = record[fieldCode]?.value || 'なし';

          const fieldElement = document.createElement('div');
          fieldElement.className = 'cardField';

          const fieldLabel = document.createElement('span');
          fieldLabel.className = 'fieldLabel';
          fieldLabel.textContent = fieldCode;

          const fieldContent = document.createElement('span');
          fieldContent.className = 'fieldContent';
          fieldContent.textContent = fieldValue;

          fieldElement.appendChild(fieldLabel);
          fieldElement.appendChild(fieldContent);
          cardContainer.appendChild(fieldElement);
        });

        textFields.forEach((fieldCode) => {
          const textBlock = document.createElement('div');
          textBlock.className = 'textBlock';

          const fieldLabel = document.createElement('span');
          fieldLabel.className = 'fieldLabel';
          fieldLabel.textContent = fieldCode;

          const fieldContent = document.createElement('span');
          fieldContent.className = 'fieldContent';
          fieldContent.textContent = record[fieldCode]?.value || 'なし';

          textBlock.appendChild(fieldLabel);
          textBlock.appendChild(fieldContent);
          cardContainer.appendChild(textBlock);
        });

        el.appendChild(cardContainer);
      }

      function formatDateTimeInJST(value) {
        if (!value) return 'なし';
        const dt = luxon.DateTime.fromISO(value, {zone: 'utc'}).setZone('Asia/Tokyo');
        return dt.isValid ? dt.toFormat('yyyy-MM-dd HH:mm') : value;
      }

      function formatUserField(recordField) {
        const users = (recordField?.value || []);
        if (users.length === 0) return 'なし';
        return users.map(u => u.name).join('・');
      }

      // シャンプー情報
      function render127Records(organizationValues) {
        const startOfDay = selectedDate.startOf('day').toFormat("yyyy-MM-dd'T'HH:mm:ssZZ");
        const endOfDay = selectedDate.endOf('day').toFormat("yyyy-MM-dd'T'HH:mm:ssZZ");

        let query127 = `開始日時 >= "${startOfDay}" and 開始日時 <= "${endOfDay}" and シャンプー in ("シャンプー有り")`;

        if (organizationValues && organizationValues.length > 0) {
          const organizationCodes = organizationValues.map(org => `"${org.code}"`).join(', ');
          query127 += ` and 利用校 in (${organizationCodes})`;
        }

        const paramsForGet127 = {
          app: APP_ID_127,
          query: query127
        };

        return kintone.api(kintone.api.url('/k/v1/records.json', true), 'GET', paramsForGet127)
          .then((resp127) => {
            const records127 = resp127.records;

            const oldShampoo = el.querySelectorAll('.shampooSection');
            oldShampoo.forEach(sec => sec.remove());

            const shampooSection = document.createElement('div');
            shampooSection.className = 'shampooSection';
            el.appendChild(shampooSection);

            const shampooHeader = document.createElement('div');
            shampooHeader.textContent = 'シャンプー情報';
            shampooHeader.className = 'shampooHeader';
            shampooSection.appendChild(shampooHeader);

            const fieldsToDisplay127 = ['氏名', '今回のお預かり対象の犬名', '開始日時', '終了日時', '利用形態', 'メモ'];

            if (records127.length > 0) {
              const tableContainer = document.createElement('div');
              tableContainer.className = 'tableContainer127';

              const table = document.createElement('table');
              table.className = 'fancyTable';

              const thead = document.createElement('thead');
              const headerRow = document.createElement('tr');
              fieldsToDisplay127.forEach((fieldName) => {
                const th = document.createElement('th');
                th.textContent = fieldName;
                headerRow.appendChild(th);
              });
              thead.appendChild(headerRow);
              table.appendChild(thead);

              const tbody = document.createElement('tbody');
              records127.forEach((record127) => {
                const row = document.createElement('tr');
                fieldsToDisplay127.forEach((fieldCode) => {
                  let displayValue = 'なし';
                  if (fieldCode === '開始日時' || fieldCode === '終了日時') {
                    displayValue = formatDateTimeInJST(record127[fieldCode]?.value);
                  } else {
                    displayValue = record127[fieldCode]?.value || 'なし';
                  }
                  const td = document.createElement('td');
                  td.textContent = displayValue;
                  row.appendChild(td);
                });
                tbody.appendChild(row);
              });
              table.appendChild(tbody);

              tableContainer.appendChild(table);
              shampooSection.appendChild(tableContainer);
            } else {
              const noRecordMessage127 = document.createElement('div');
              noRecordMessage127.textContent = '条件に合致するAPP_ID_127のレコードはありません。';
              noRecordMessage127.className = 'noRecordMessage127';
              shampooSection.appendChild(noRecordMessage127);
            }

            return Promise.resolve(organizationValues);
          })
          .catch((error) => {
            console.error(error);
            alert('APP_ID_127のデータ取得中にエラーが発生しました。');
            return Promise.resolve(organizationValues);
          });
      }

      // 来校情報
      function renderAllRaikouRecords(organizationValues) {
        const raikouSection = document.createElement('div');
        raikouSection.className = 'raikouSection';
        el.appendChild(raikouSection);

        const raikouHeader = document.createElement('div');
        raikouHeader.textContent = '来校情報';
        raikouHeader.className = 'raikouHeader';
        raikouSection.appendChild(raikouHeader);

        return renderRaikouRecordsForCar(organizationValues, "１号車", raikouSection)
          .then(() => {
            return renderRaikouRecordsForCar(organizationValues, "２号車", raikouSection);
          })
          .then(() => {
            return renderRaikouRecordsForCar(organizationValues, "３号車", raikouSection);
          });
      }

      function renderRaikouRecordsForCar(organizationValues, carName, raikouSection) {
        const startOfDay = selectedDate.startOf('day').toFormat("yyyy-MM-dd'T'HH:mm:ssZZ");
        const endOfDay = selectedDate.endOf('day').toFormat("yyyy-MM-dd'T'HH:mm:ssZZ");
        let queryRaikou = `開始日時 >= "${startOfDay}" and 開始日時 <= "${endOfDay}" and 利用する送迎車_迎え in ("${carName}")`;

        if (organizationValues && organizationValues.length > 0) {
          const organizationCodes = organizationValues.map(org => `"${org.code}"`).join(', ');
          queryRaikou += ` and 利用校 in (${organizationCodes})`;
        }

        const paramsForRaikou = {
          app: APP_ID_127,
          query: queryRaikou
        };

        return kintone.api(kintone.api.url('/k/v1/records.json', true), 'GET', paramsForRaikou)
          .then((respRaikou) => {
            const recordsRaikou = respRaikou.records;

            const raikouSubHeader = document.createElement('div');
            raikouSubHeader.textContent = `お迎え-${carName}`;
            raikouSubHeader.className = 'raikouSubHeader';
            raikouSection.appendChild(raikouSubHeader);

            const fieldsToDisplayRaikou = [
              '開始日時', '終了日時', '氏名', '今回のお預かり対象の犬名',
              '利用形態', '来校手段', '利用する送迎車_迎え', '送迎担当者_迎え', 'メモ'
            ];

            if (recordsRaikou.length > 0) {
              const tableContainer = document.createElement('div');
              tableContainer.className = 'tableContainer127';

              const table = document.createElement('table');
              table.className = 'fancyTable';

              const thead = document.createElement('thead');
              const headerRow = document.createElement('tr');
              fieldsToDisplayRaikou.forEach((fieldName) => {
                const th = document.createElement('th');
                th.textContent = fieldName;
                headerRow.appendChild(th);
              });
              thead.appendChild(headerRow);
              table.appendChild(thead);

              const tbody = document.createElement('tbody');
              recordsRaikou.forEach((recordRaikou) => {
                const row = document.createElement('tr');
                fieldsToDisplayRaikou.forEach((fieldCode) => {
                  let displayValue = 'なし';
                  if (fieldCode === '開始日時' || fieldCode === '終了日時') {
                    displayValue = formatDateTimeInJST(recordRaikou[fieldCode]?.value);
                  } else if (fieldCode === '送迎担当者_迎え') {
                    displayValue = formatUserField(recordRaikou[fieldCode]);
                  } else {
                    displayValue = recordRaikou[fieldCode]?.value || 'なし';
                  }
                  const td = document.createElement('td');
                  td.textContent = displayValue;
                  row.appendChild(td);
                });
                tbody.appendChild(row);
              });
              table.appendChild(tbody);

              tableContainer.appendChild(table);
              raikouSection.appendChild(tableContainer);
            } else {
              const noRecordMessage = document.createElement('div');
              noRecordMessage.textContent = '条件に合致するAPP_ID_127のレコードはありません。';
              noRecordMessage.className = 'noRecordMessage127';
              raikouSection.appendChild(noRecordMessage);
            }
          })
          .catch((error) => {
            console.error(error);
            alert('APP_ID_127の来校情報取得中にエラーが発生しました。');
          });
      }

      // 退校情報
      function renderAllTaikouRecords(organizationValues) {
        const taikouSection = document.createElement('div');
        taikouSection.className = 'taikouSection';
        el.appendChild(taikouSection);

        const taikouHeader = document.createElement('div');
        taikouHeader.textContent = '退校情報';
        taikouHeader.className = 'raikouHeader';
        taikouSection.appendChild(taikouHeader);

        return renderTaikouRecordsForCar(organizationValues, "１号車", taikouSection)
          .then(() => {
            return renderTaikouRecordsForCar(organizationValues, "２号車", taikouSection);
          })
          .then(() => {
            return renderTaikouRecordsForCar(organizationValues, "３号車", taikouSection);
          });
      }

      function renderTaikouRecordsForCar(organizationValues, carName, taikouSection) {
        const startOfDay = selectedDate.startOf('day').toFormat("yyyy-MM-dd'T'HH:mm:ssZZ");
        const endOfDay = selectedDate.endOf('day').toFormat("yyyy-MM-dd'T'HH:mm:ssZZ");
        let queryTaikou = `終了日時 >= "${startOfDay}" and 終了日時 <= "${endOfDay}" and 利用する送迎車_送り in ("${carName}")`;

        // 来校情報と同様に、利用校が複数組織を持つ組織選択フィールドなので in (...)で絞り込む
        if (organizationValues && organizationValues.length > 0) {
          const organizationCodes = organizationValues.map(org => `"${org.code}"`).join(', ');
          queryTaikou += ` and 利用校 in (${organizationCodes})`;
        }

        const paramsForTaikou = {
          app: APP_ID_127,
          query: queryTaikou
        };

        return kintone.api(kintone.api.url('/k/v1/records.json', true), 'GET', paramsForTaikou)
          .then((respTaikou) => {
            const recordsTaikou = respTaikou.records;

            const taikouSubHeader = document.createElement('div');
            taikouSubHeader.textContent = `送り-${carName}`;
            taikouSubHeader.className = 'raikouSubHeader';
            taikouSection.appendChild(taikouSubHeader);

            const fieldsToDisplayTaikou = [
              '開始日時', '終了日時', '氏名', '今回のお預かり対象の犬名',
              '利用形態', '来校手段', '利用する送迎車_送り', '送迎担当者_送り', 'メモ'
            ];

            if (recordsTaikou.length > 0) {
              const tableContainer = document.createElement('div');
              tableContainer.className = 'tableContainer127';

              const table = document.createElement('table');
              table.className = 'fancyTable';

              const thead = document.createElement('thead');
              const headerRow = document.createElement('tr');
              fieldsToDisplayTaikou.forEach((fieldName) => {
                const th = document.createElement('th');
                th.textContent = fieldName;
                headerRow.appendChild(th);
              });
              thead.appendChild(headerRow);
              table.appendChild(thead);

              const tbody = document.createElement('tbody');
              recordsTaikou.forEach((recordTaikou) => {
                const row = document.createElement('tr');
                fieldsToDisplayTaikou.forEach((fieldCode) => {
                  let displayValue = 'なし';
                  if (fieldCode === '開始日時' || fieldCode === '終了日時') {
                    displayValue = formatDateTimeInJST(recordTaikou[fieldCode]?.value);
                  } else if (fieldCode === '送迎担当者_送り') {
                    displayValue = formatUserField(recordTaikou[fieldCode]);
                  } else {
                    displayValue = recordTaikou[fieldCode]?.value || 'なし';
                  }
                  const td = document.createElement('td');
                  td.textContent = displayValue;
                  row.appendChild(td);
                });
                tbody.appendChild(row);
              });
              table.appendChild(tbody);

              tableContainer.appendChild(table);
              taikouSection.appendChild(tableContainer);
            } else {
              const noRecordMessage = document.createElement('div');
              noRecordMessage.textContent = '条件に合致するAPP_ID_127のレコードはありません。';
              noRecordMessage.className = 'noRecordMessage127';
              taikouSection.appendChild(noRecordMessage);
            }
          })
          .catch((error) => {
            console.error(error);
            alert('APP_ID_127の退校情報取得中にエラーが発生しました。');
          });
      }

      function updateData() {
        updateDateDisplay();

        const targetDate = selectedDate.toFormat('yyyy-MM-dd');
        const query164 = `対象日 = "${targetDate}"`;
        const paramsForGet164 = { app: APP_ID_164, query: query164 };

        kintone.api(kintone.api.url('/k/v1/records.json', true), 'GET', paramsForGet164)
          .then((resp) => {
            el.innerHTML = '';
            el.appendChild(dateDisplay);

            const records = resp.records;
            currentRecords164 = records;
            organizationDisplay.innerHTML = '';

            if (records.length === 0) {
              organizationDisplay.textContent = '未設定';
              render164Records(null);
              render127Records([]).then((orgVals) => {
                return renderAllRaikouRecords(orgVals);
              }).then((orgVals2) => {
                return renderAllTaikouRecords(orgVals2);
              });
            } else if (records.length === 1) {
              const record = records[0];
              const organizationValues = record['組織']?.value || [];
              organizationDisplay.textContent = organizationValues.map(org => org.name).join('・') || '未設定';
              render164Records(record);
              render127Records(organizationValues).then((orgVals) => {
                return renderAllRaikouRecords(orgVals);
              }).then((orgVals2) => {
                return renderAllTaikouRecords(orgVals2);
              });
            } else {
              const select = document.createElement('select');
              const defaultOption = document.createElement('option');
              defaultOption.value = '';
              defaultOption.textContent = '組織を選択';
              select.appendChild(defaultOption);

              records.forEach((record) => {
                const orgVals = record['組織']?.value || [];
                const orgNames = orgVals.map(org => org.name).join('・') || '未設定';
                const option = document.createElement('option');
                option.value = record.$id.value;
                option.textContent = `ID:${record.$id.value} / ${orgNames}`;
                select.appendChild(option);
              });

              organizationDisplay.appendChild(select);

              select.addEventListener('change', () => {
                const selectedId = select.value;
                const selectedRecord = currentRecords164.find(r => r.$id.value === selectedId);
                if (selectedRecord) {
                  const orgVals = selectedRecord['組織']?.value || [];
                  render164Records(selectedRecord);
                  render127Records(orgVals).then((orgVals2) => {
                    return renderAllRaikouRecords(orgVals2);
                  }).then((orgVals3) => {
                    return renderAllTaikouRecords(orgVals3);
                  });
                } else {
                  render164Records(null);
                  render127Records([]).then((orgVals4) => {
                    return renderAllRaikouRecords(orgVals4);
                  }).then((orgVals5) => {
                    return renderAllTaikouRecords(orgVals5);
                  });
                }
              });

              // 初期状態
              render164Records(null);
              render127Records([]).then((orgVals6) => {
                return renderAllRaikouRecords(orgVals6);
              }).then((orgVals7) => {
                return renderAllTaikouRecords(orgVals7);
              });
            }
            return resp;
          })
          .catch((error) => {
            console.error(error);
            alert('データ取得中にエラーが発生しました。');
          });
      }

      // 初期化
      updateData();

      nextButton.addEventListener('click', () => {
        selectedDate = selectedDate.plus({ days: 1 });
        updateData();
      });

      prevButton.addEventListener('click', () => {
        selectedDate = selectedDate.minus({ days: 1 });
        updateData();
      });

      return event;
    });
})();
