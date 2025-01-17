<template>
    <v-card>
        <v-card-title> 导入图书 <v-chip small class="primary">Beta</v-chip> </v-card-title>
        <v-card-text>
        请将需要导入的书籍放入{{ scan_dir }}目录中。 支持的格式为 azw/azw3/epub/mobi/pdf/txt 。<br/>
        请注意：此功能为后台异步执行，不必重复点击，可刷新关注表格状态进展。已导入成功的记录请不要删除，以免书籍被再次导入。<br/>
        另外，还可以使用<a target="_blank" href="https://calibre-ebook.com/">PC版Calibre软件</a>管理书籍，但是请注意：使用完PC版后，需重启Web版方可生效。
        </v-card-text>
        <v-card-actions>
            <v-btn :disabled="loading" color="primary" @click="scan_books"><v-icon>mdi-file-find</v-icon>扫描书籍</v-btn>
            <v-btn :disabled="loading" outlined color="primary" @click="getDataFromApi"><v-icon>mdi-reload</v-icon>刷新</v-btn>
            <template v-if="selected.length > 0">
                <v-btn :disabled="loading" outlined color="primary" @click="import_books"><v-icon>mdi-import</v-icon>导入书籍 </v-btn>
                <v-btn :disabled="loading" outlined color="primary" @click="delete_record"><v-icon>mdi-delete</v-icon>删除 </v-btn>
            </template>
        </v-card-actions>
        <v-card-text>
            <div v-if="selected.length == 0">请勾选需要处理的文件</div>
            <div v-else>共选择了{{ selected.length }}个</div>
        </v-card-text>
        <v-data-table
            dense
            class="elevation-1 text-body-2"
            ref="dataTable"
            show-select
            v-model="selected"
            item-key="hash"
            :search="search"
            :headers="headers"
            :items="items"
            :options.sync="options"
            :server-items-length="total"
            :loading="loading"
            :page.sync="page"
            :items-per-page.sync="perPage"
            :footer-props="{ 'items-per-page-options': [20, 50, 100, 1000] }"
        >
            <template v-slot:item.status="{ item }">
                <v-chip small v-if="item.status == 'ready'" class="success">可导入</v-chip>
                <v-chip small v-else-if="item.status == 'exist'" class="lighten-4">已存在</v-chip>
                <v-chip small v-else-if="item.status == 'imported'" class="primary">导入成功</v-chip>
                <v-chip small v-else-if="item.status == 'new'" class="grey">待扫描</v-chip>
                <v-chip small v-else class="info">{{ item.status }}</v-chip>
            </template>
            <template v-slot:item.title="{ item }">
                书名：<span v-if="item.book_id == 0"> {{ item.title }} </span>
                <a v-else target="_blank" :href="`/book/${item.book_id}`">{{ item.title }}</a> <br />
                作者：{{ item.author }}
            </template>
            <template v-slot:footer >
                <div style="display: flex; flex-flow: row; margin-left: 8px; margin-right: 8px" >
                    
                        <v-select
                            style="margin-top: 24px; margin-left: 8px; width: 72px; flex: 0 1 0"
                            :value="perPage"
                            :items="[20, 50, 100, 1000]"
                            density="compact"
                            variant="solo"
                            @change="handlePageSelectChange"
                            ></v-select>
                        <div style="display: flex; align-items: center; justify-content: center; flex: 1 1 0" >
                            <v-pagination 
                                v-model="page" 
                                :length="total" 
                                total-visible="12" 
                                style="margin: 0"
                                circle ></v-pagination>
                        </div>
                        <!-- length：页码的总长度；total-visible：显示的最大可见分页数 -->
                    <!-- 加了下面的部分👇 -->
                        <div style="display: flex; align-items: center; justify-content: end; padding-right: 12px" >
                            <div>跳转</div>
                            <v-text-field solo v-model="whichPage" type="number" 
                                style="margin-top: 24px; width: 80px; margin-start: 4px; margin-end: 4px"
                                @keyup.enter="goToPage"></v-text-field>
                            <div>页</div>
                            <v-btn small fab style="display: none" @click="goToPage()">GO</v-btn>
                        </div>
                </div>
            </template>
        </v-data-table>
            
    </v-card>
</template>

<script>
export default {
    data: () => ({
        selected: [],
        scan_dir: "/data/books/imports/",
        search: "",
        page: 1,
        perPage: 100,
        items: [],
        total: 0,
        loading: false,
        options: {
            page: 1,
            itemsPerPage: 100,
        },
        headers: [
            { text: "ID", sortable: true, value: "id" },
            { text: "状态", sortable: true, value: "status" },
            { text: "路径", sortable: true, value: "path" },
            { text: "扫描信息", sortable: false, value: "title" },
            { text: "时间", sortable: true, value: "create_time", width: "200px" },
        ],
        progress: {
            done: 0,
            total: 0,
            status: "finish",
        },
        whichPage: 1
    }),
    watch: {
        options: {
            handler() {
                this.getDataFromApi();
            },
            deep: true,
        },
    },
    mounted() {
        this.getDataFromApi();
        this.total = 100
    },
    computed: {
        pageCount: function () {
            return parseInt(this.total / 20);
        },
    },
    methods: {
        handlePageSelectChange(newValue) {
            console.log(`value: ${newValue}`)
            console.log(`table: ${this.$refs.dataTable}`)
            // this.$refs.dataTable.setItemsPerPage(Number(newValue))
            this.perPage = Number(newValue)
            this.options.itemsPerPage = Number(newValue)
        },
        goToPage() {
            console.log(`goToPage: ${this.whichPage}`)
            this.page = Number(this.whichPage)
        },
        getDataFromApi() {
            console.log(`getDataFromApi`)
            this.loading = true;
            const { sortBy, sortDesc, page, itemsPerPage } = this.options;

            var data = new URLSearchParams();
            if (page != undefined) {
                data.append("page", page);
            }
            if (sortBy != undefined) {
                data.append("sort", sortBy);
            }
            if (sortDesc != undefined) {
                data.append("desc", sortDesc);
            }
            if (itemsPerPage != undefined) {
                data.append("num", itemsPerPage);
            }
            console.log(`data: ${data.toString()}`)
            this.$backend("/admin/scan/list?" + data.toString())
                .then((rsp) => {
                    if (rsp.err != "ok") {
                        this.items = [];
                        this.total = 0;
                        alert(rsp.msg);
                        return false;
                    }
                    // this.items = rsp.items;
                    // this.total = rsp.total;
                    // this.scan_dir = rsp.scan_dir;
                })
                .finally(() => {
                    this.loading = false;
                });
        },
        loop_check_status(url, callback) {
            this.$backend(url)
                .then((rsp) => {
                    if (rsp.err != "ok") {
                        this.$alert("error", rsp.msg);
                        return;
                    }
                    if (callback(rsp)) {
                        setTimeout(() => {
                            this.loop_check_status(url, callback);
                        }, 1000);
                    } else {
                        this.getDataFromApi();
                        this.$alert("info", "处理完毕！");
                    }
                })
        },
        scan_books() {
            this.loading = true;
            this.$backend("/admin/scan/run", {
                method: "POST",
            })
                .then((rsp) => {
                    if (rsp.err !== "ok") {
                        this.$alert("error", rsp.msg);
                        return;
                    }

                    //this.check_scan_status();
                    this.loop_check_status("/admin/scan/status", (rsp) => {
                        this.scan = rsp.status;
                        if (this.scan.new === 0) {
                            this.loading = false;
                            return false;
                        }
                        this.loading = true;
                        return true;
                    });
                })
                .finally(() => {});
        },
        import_books() {
            this.loading = true;
            this.$backend("/admin/import/run", {
                method: "POST",
                body: JSON.stringify({
                    hashlist: this.selected.map((v) => {
                        return v.hash;
                    }),
                }),
            })
                .then((rsp) => {
                    if (rsp.err !== "ok") {
                        this.$alert("error", rsp.msg);
                    }
                    //this.check_import_status();
                    this.loop_check_status("/admin/import/status", (rsp) => {
                        this.import = rsp.status;
                        if (this.import.ready === 0) {
                            this.loading = false;
                            return false;
                        }
                        this.loading = true;
                        return true;
                    });
                })
                .finally(() => {
                    this.loading = false;
                });
        },
        delete_record() {
            console.log(this.selected);
            this.loading = true;
            this.$backend("/admin/scan/delete", {
                method: "POST",
                body: JSON.stringify({
                    hashlist: this.selected.map((v) => {
                        return v.hash;
                    }),
                }),
            })
                .then((rsp) => {
                    if (rsp.err !== "ok") {
                        this.$alert("error", rsp.msg);
                    }
                    this.getDataFromApi();
                })
                .finally(() => {
                    this.loading = false;
                });
        },
        mark_as(status) {
            this.loading = true;
            this.$backend("/admin/scan/mark", {
                method: "POST",
                body: JSON.stringify({ hashlist: this.selected, status: status }),
            })
                .then((rsp) => {
                    if (rsp.err !== "ok") {
                        this.$alert("error", rsp.msg);
                    }
                })
                .finally(() => {
                    this.loading = false;
                });
            this.items.map((v) => {
                if (this.selected.indexOf(v.hash)) {
                    v.status = status;
                }
            });
        },
    },
};
</script>
