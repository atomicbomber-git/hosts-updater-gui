<script lang="tsx">
import {debounce} from "lodash"
import {defineComponent, reactive, ref, computed} from "vue"
import readline from "readline"
import fs from "fs"
import StringSimilarity from "string-similarity"

interface Plain {
    value: string
}

interface Entry {
    ipAddress: string,
    domains: string[],
}

type LineModel = Plain | Entry

function stringToLineModel(line: string): LineModel {
    line = line.trim()
    
    if (line.startsWith(`#`) || (line.length === 0)) {
        return {value: line}
    }
    
    const lineParts = line.split(/ +/ui)
    return {
        ipAddress: lineParts[0],
        domains: lineParts.splice(1),
    }
}

function isEntry(object: unknown): object is Entry {
    return Array.isArray((object as Entry).domains) &&
        (object as Entry).ipAddress !== "undefined"
}

function fetchLineModels(lineModels: LineModel[]) {
    readline.createInterface({
        input: fs.createReadStream(`/etc/hosts`)
    }).on(`line`, lineString => {
        /* Process lineString one by one */
        lineModels.push(stringToLineModel(lineString))
    }).on(`close`, () => {
        /* What happens after the last line has been read */
    })
}

export default defineComponent({
    setup() {
        let lineModels = reactive<LineModel[]>([])
        let searchString = ref<string|undefined>(undefined)
        
        fetchLineModels(lineModels);
        
        const sortedEntries = computed(() =>
            lineModels
                .filter(isEntry)
                .sort((leftEntry: Entry, rightEntry: Entry) => {
                        const bestMatchRatingA = Math.max(
                            ...StringSimilarity.findBestMatch( searchString.value ?? "", leftEntry.domains)
                                .ratings
                                .map(rating => rating.rating)
                        )

                        const bestMatchRatingB = Math.max(
                            ...StringSimilarity.findBestMatch( searchString.value ?? "", rightEntry.domains)
                                .ratings
                                .map(rating => rating.rating)
                        )

                        return bestMatchRatingB - bestMatchRatingA
                })
        )
        
        return {
            searchString,
            lineModels,
            sortedLineModels: sortedEntries,
            onSearchStringInput: debounce((e: Event) => {
                searchString.value = (e.target as HTMLInputElement).value
            }, 500)
        }
    },
    
    render() {
        return (
            <article>
                <div class="container">
                    <div class="fixed-top container">
                        <div class="card my-2">
                            <div class="card-body">
                                <input
                                    value={this.searchString}
                                    onInput={this.onSearchStringInput}
                                    class="form-control"
                                    placeholder="Search"
                                    type="text"
                                />
                            </div>
                        </div>
                    </div>
                    
                    <div class="list-group" style="margin-top: 100px">
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
                                            <button class="btn btn-danger btn-sm">
                                                Delete
                                            </button>
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