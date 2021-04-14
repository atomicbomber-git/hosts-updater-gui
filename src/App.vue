<script lang="tsx">
import {debounce} from "lodash"
import { ipcRenderer, remote } from "electron"
import {Modal} from "bootstrap"
import {computed, defineComponent, onMounted, reactive, ref} from "vue"
import readline from "readline"
import fs from "fs"
import StringSimilarity from "string-similarity"
import axios from "axios"

interface Plain {
    id: number,
    value: string
    isDeleted: boolean,
}

interface Entry {
    id: number,
    ipAddress: string,
    domains: string[],
    isDeleted: boolean,
}

type LineModel = Plain | Entry


function getHostsFilePath(): string {
    return "/etc/hosts"
}

function getTempHostsFilePath(): string {
    return `${remote.app.getAppPath()}/hosts.temp`
}

const getNextId: () => number = (function () {
    let internalCounter = 0
    return () => internalCounter++
})()

function stringToLineModel(line: string, getNextNumber: () =>  number): LineModel {
    line = line.trim()
    
    if (line.startsWith(`#`) || (line.length === 0)) {
        return { id: getNextNumber(), isDeleted: false,  value: line } as Plain
    }
    
    const lineParts = line.split(/ +/ui)
    return {
        id: getNextNumber(),
        isDeleted: false,
        ipAddress: lineParts[0],
        domains: lineParts.splice(1),
    } as Entry
}

function isEntry(object: unknown): object is Entry {
    return Array.isArray((object as Entry).domains) &&
        (object as Entry).ipAddress !== "undefined"
}

async function flushFile(lineModels: LineModel[]) {
    const tempFilePath = getTempHostsFilePath()
    
    fs.writeFileSync(
        tempFilePath,
        lineModels.map(renderLineModel).join(`\n`)
    )
    
    ipcRenderer.send("save", {
        src: tempFilePath,
        dest: getHostsFilePath(),
    })
}

function renderLineModel(lineModel: LineModel): string {
    if (isEntry(lineModel)) {
        return `${lineModel.ipAddress} ${lineModel.domains.join(' ')}`
    } else {
        return lineModel.value
    }
}

interface ApiResponse {
    ip_addresses: string[],
    message: string,
    status: number,
}

function fetchLineModels(): Promise<LineModel[]> {
    return new Promise<LineModel[]>(resolve => {
        const lineModels: LineModel[] = []
    
        readline.createInterface({
            input: fs.createReadStream(`/etc/hosts`)
        }).on(`line`, lineString => {
            /* Process lineString one by one */
            lineModels.push(stringToLineModel(lineString, getNextId))
        }).on(`close`, () => {
            resolve(lineModels)
        })
    })
}

function sortLineModelsBySearchString(searchString: string) {
    return (leftEntry: Entry, rightEntry: Entry) => {
        const bestMatchRatingA = Math.max(
            ...StringSimilarity.findBestMatch(searchString ?? "", leftEntry.domains)
                .ratings
                .map(rating => rating.rating)
        )
        
        const bestMatchRatingB = Math.max(
            ...StringSimilarity.findBestMatch(searchString ?? "", rightEntry.domains)
                .ratings
                .map(rating => rating.rating)
        )
        
        return bestMatchRatingB - bestMatchRatingA
    };
}

export default defineComponent({
    setup() {
        let lineModels = reactive<LineModel[]>([])
        let searchString = ref<string | undefined>(undefined)
        
        fetchLineModels().then(obtainedLineModels => {
            lineModels.push(...obtainedLineModels)
        })
        
        const sortedEntries = computed(() =>
            lineModels
                .filter(isEntry)
                .sort(sortLineModelsBySearchString(searchString.value ?? ""))
        )
        
        const modalRef = ref<HTMLDivElement>()
        let modal: Modal | null = null
        
        onMounted(() => {
            if (typeof modalRef.value !== "undefined") {
                modal = new Modal(modalRef.value)
            }
        })
        
        const onDeleteEntryButtonClick = (entry: Entry) => {
            entry.isDeleted = true
        }
    
        const onRestoreEntryButtonClick = (entry: Entry) => {
            entry.isDeleted = false
        }
        
        return {
            searchString,
            lineModels,
            sortedLineModels: sortedEntries,
            modalRef,
            
            onRenewEntryButtonClick (entry: Entry) {
                Promise.all(entry.domains.map(async (domain: string) => {
                    return await axios.get<ApiResponse>(`https://addr-scraper.jamespatrickkeegan.com/?domain=${domain}`)
                })).then(responses => {
                     if (responses.length > 0) {
                         let firstResponse = responses[0].data
                         entry.ipAddress = firstResponse.ip_addresses[0]
                     }
                }).catch(error => {
                    console.log(error)
                    alert("Failed to contact server.")
                })
            },
            
            renderDeleteOrRestoreButton(entry: Entry) {
                if (entry.isDeleted) {
                    return (
                        <button onClick={() => { onRestoreEntryButtonClick(entry) }} class="btn btn-success btn-sm">
                            Restore
                        </button>
                    )
                } else {
                    return (
                        <button onClick={() => { onDeleteEntryButtonClick(entry) }} class="btn btn-danger btn-sm">
                            Delete
                        </button>
                    )
                }
            },
            
            onPreviewButtonClick() {
                modal?.toggle()
            },
            
            onSearchStringInput: debounce((e: Event) => {
                searchString.value = (e.target as HTMLInputElement).value
            }, 500)
        }
    },
    
    render() {
        return (
            <article>
                <div class="modal"
                     tabindex="-1"
                     ref="modalRef"
                >
                    <div class="modal-dialog modal-xl">
                        <div class="modal-content">
                            <div class="modal-header">
                                <h5 class="modal-title"> Preview Of The File </h5>
                                <button type="button"
                                        class="btn-close"
                                        data-bs-dismiss="modal"
                                        aria-label="Close"
                                />
                            </div>
                            
                            <div class="modal-body">
                                <div class="list-group">
                                    {this.lineModels.map(lineModel => (
                                        <div class="list-group-item list-group-item-action">
                                            {renderLineModel(lineModel)}
                                        </div>
                                    ))}
                                </div>
                            </div>
                            
                            <div class="modal-footer">
                                <button type="button"
                                        class="btn btn-secondary"
                                        data-bs-dismiss="modal"
                                >Close
                                </button>
                                <button type="button"
                                        class="btn btn-primary"
                                >Save changes
                                </button>
                            </div>
                        </div>
                    </div>
                </div>
                
                <div class="container">
                    <div class="fixed-top container">
                        <div class="card my-2">
                            <div class="card-body d-flex">
                                <input
                                    value={this.searchString}
                                    onInput={this.onSearchStringInput}
                                    class="form-control"
                                    placeholder="Search"
                                    type="text"
                                />
                                
                                <button
                                    onClick={this.onPreviewButtonClick}
                                    class="btn btn-primary ms-3"
                                >
                                    Preview
                                </button>
                            </div>
                        </div>
                    </div>
                    
                    <div class="list-group"
                         style="margin-top: 100px"
                    >
                        {this.sortedLineModels.map(lineModel =>
                            isEntry(lineModel) && (
                                <div class="list-group-item list-group-item-action">
                                    <div class="row align-items-end">
                                        <div class="col">
                                            <h2 class="h5"> {lineModel.ipAddress} </h2>
                                            <div>
                                                {lineModel.domains.map(domain => (
                                                        <span class="badge bg-primary me-2">
                                                {domain}
                                            </span>
                                                    )
                                                )}
                                            </div>
                                        </div>
                                        <div class="col text-end">
                                            <button onClick={() => { this.onRenewEntryButtonClick(lineModel) }} class="btn btn-info btn-sm">
                                                Renew
                                            </button>
                                            
                                            {this.renderDeleteOrRestoreButton(lineModel)}
                                        </div>
                                    </div>
                                </div>
                            )
                        )}
                    </div>
                </div>
            </article>
        )
    }
})
</script>

<style src="bootstrap/dist/css/bootstrap.css"/>