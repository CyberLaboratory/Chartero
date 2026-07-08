<script lang="ts">
import type { AttachmentHistory } from '$/history/history';
import { toTimeString } from '$/utils';

interface ReadRow {
    key: string;
    date: string;
    sortTime: number;
    itemTitle: string;
    attachmentTitle: string;
    pages: number[];
    pageLabel: string;
    totalSeconds: number;
    openItemID: number;
    openPageIndex: number;
}

function compressPages(pages: number[]) {
    if (!pages.length) return '';

    const sorted = [...new Set(pages)].sort((a, b) => a - b);
    const ranges: string[] = [];
    let start = sorted[0];
    let prev = sorted[0];

    for (let i = 1; i < sorted.length; ++i) {
        const page = sorted[i];
        if (page === prev + 1) {
            prev = page;
            continue;
        }
        ranges.push(start === prev ? String(start + 1) : `${start + 1}-${prev + 1}`);
        start = prev = page;
    }

    ranges.push(start === prev ? String(start + 1) : `${start + 1}-${prev + 1}`);
    return ranges.join(', ');
}

export default {
    props: {
        history: {
            type: Array<AttachmentHistory>,
            required: true,
        },
    },
    data() {
        return {
            locale: addon.locale,
            rows: [] as ReadRow[],
        };
    },
    watch: {
        history: {
            immediate: true,
            handler(newHis: AttachmentHistory[]) {
                const grouped = new Map<string, Omit<ReadRow, 'key' | 'pageLabel'>>();

                newHis.forEach(his => {
                    const attachment = Zotero.Items.getByLibraryAndKey(his.note.libraryID, his.key);
                    if (!attachment) return;

                    const itemTitle = attachment.parentItem
                        ? String(attachment.parentItem.getField('title'))
                        : String(attachment.getField('title'));
                    const attachmentTitle = String(attachment.getField('title'));

                    Object.entries(his.record.pages).forEach(([pageIndex, page]) => {
                        if (!page.period) return;

                        Object.entries(page.period).forEach(([timestamp, seconds]) => {
                            const date = new Date(Number(timestamp) * 1000);
                            const dateLabel = date.toLocaleDateString(Zotero.locale, {
                                year: 'numeric',
                                month: 'short',
                                day: 'numeric',
                            });
                            const key = `${attachment.id}:${dateLabel}`;
                            const row = grouped.get(key) ?? {
                                date: dateLabel,
                                sortTime: new Date(
                                    date.getFullYear(),
                                    date.getMonth(),
                                    date.getDate()
                                ).getTime(),
                                itemTitle,
                                attachmentTitle,
                                pages: [],
                                totalSeconds: 0,
                                openItemID: attachment.id,
                                openPageIndex: Number(pageIndex),
                            };
                            row.pages.push(Number(pageIndex));
                            row.totalSeconds += Number(seconds);
                            row.openPageIndex = Math.min(row.openPageIndex, Number(pageIndex));
                            grouped.set(key, row);
                        });
                    });
                });

                this.rows = [...grouped.entries()]
                    .map(([key, row]) => ({
                        ...row,
                        key,
                        pageLabel: compressPages(row.pages),
                    }))
                    .sort((a, b) => b.sortTime - a.sortTime || a.itemTitle.localeCompare(b.itemTitle));
            },
        },
    },
    computed: {
        hasRows() {
            return this.rows.length > 0;
        },
    },
    methods: {
        openRow(row: ReadRow) {
            Zotero.FileHandlers.open(Zotero.Items.get(row.openItemID), {
                location: { pageIndex: row.openPageIndex },
            });
        },
    },
    setup() {
        return { toTimeString };
    },
};
</script>

<template>
  <div v-if="hasRows" class="table-wrap">
    <table class="read-table">
      <thead>
        <tr>
          <th>{{ locale.date }}</th>
          <th>{{ locale.itemTitle }}</th>
          <th>{{ locale.fileName }}</th>
          <th>{{ locale.pages }}</th>
          <th>{{ locale.totalTime }}</th>
        </tr>
      </thead>
      <tbody>
        <tr
          v-for="row of rows" :key="row.key" class="read-row" tabindex="0"
          @dblclick="openRow(row)" @keydown.enter="openRow(row)"
        >
          <td>{{ row.date }}</td>
          <td :title="row.itemTitle">{{ row.itemTitle }}</td>
          <td :title="row.attachmentTitle">{{ row.attachmentTitle }}</td>
          <td>{{ row.pageLabel }}</td>
          <td>{{ toTimeString(row.totalSeconds) }}</td>
        </tr>
      </tbody>
    </table>
  </div>
  <h1 v-else class="center-label">
    {{ locale.noHistoryFound }}
  </h1>
</template>

<style scoped>
.table-wrap {
    padding: 8px;
    max-height: 420px;
    overflow: auto;
}

.read-table {
    width: 100%;
    border-collapse: collapse;
    table-layout: fixed;
    font-size: 12px;
}

.read-table th,
.read-table td {
    padding: 8px 10px;
    border-bottom: 1px solid var(--td-component-stroke, #dcdcdc);
    text-align: left;
    vertical-align: top;
    word-break: break-word;
}

.read-table thead th {
    position: sticky;
    top: 0;
    background: var(--td-bg-color-container, #fff);
    z-index: 1;
}

.read-row {
    cursor: pointer;
}

.read-row:hover,
.read-row:focus-visible {
    background: var(--td-bg-color-container-hover, #f5f5f5);
    outline: none;
}

.center-label {
    text-align: center;
    padding: 24px 12px;
}
</style>
