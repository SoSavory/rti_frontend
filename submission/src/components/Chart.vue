<script setup>
    import { ref, onMounted, computed, watch } from 'vue'
    import * as d3 from 'd3'
    
    const DATA_URL = `https://raw.githubusercontent.com/rtidatascience/frontend-dev-exercises/master/census.cs`
    
    const data = ref(null)

    // Dynamic axis from csv data ultimately displayed by the chart vs. "% over_50k"
    // Current options are: "education_level" and "race"
    const activeAxis = ref("education_level")

    // filter states
    const filterAgeGT = ref(null)
    const filterAgeLT = ref(null)
    const filterSex   = ref(null)

    // Cached/computed view of data after it has been (optionally) filtered and transformed. 
    // This is fed into a d3 chart
    const chartData = computed(() => {
        let filteredData = data.value
        if(filteredData){
            if(filterAgeGT.value){
                filteredData = filteredData.filter((datum) => datum.age >= filterAgeGT.value)
            }
            if(filterAgeLT.value){
                filteredData = filteredData.filter((datum) => datum.age <= filterAgeLT.value)
            }
            if(filterSex.value){
                filteredData = filteredData.filter((datum) => datum.sex == filterSex.value)
            }
        }
        return transformData(filteredData, axisReducer)
    })

    // Tries to fetch census.csv from github if possible, otherwise, loads in a locally stored copy
    const fetchCSV = async () => {
        data.value = null
        try {
            const response = await fetch(DATA_URL)
            if(!response.ok){
                throw new Error(`${response.status}`)
            }
            data.value = d3.csvParse(await response.text())
        } catch(error) {
            console.error('Error fetching data: ', error)
            console.error('Using local backup')
            
            data.value = await d3.csv('/census.csv')
        }
    }

    // const raceReducer = (accum, {race, over_50k}) => {
    //     accum[race] = accum[race] || [0,0]
    //     accum[race][0] += parseInt(over_50k) ? 1 : 0
    //     accum[race][1]++
    //     return accum
    // }

    // const edReducer = (accum, {education_level, over_50k}) => {
    //     accum[education_level] = accum[education_level] || [0,0]
    //     accum[education_level][0] += parseInt(over_50k) ? 1 : 0 
    //     accum[education_level][1]++
    //     return accum 
    // }

    // reduces objectified csv data into a dict like: {high_school: [# over 50k, # total], college: [30, 100], ...}
    // Used as an argument for transformData
    const axisReducer = (accum, curr) => {
        // dynamically access key from csv-data-object 
        const a  = curr[activeAxis.value]
        // initialize value from csv as key for output dict 
        // with [numerator, denominator] as value
        accum[a] = accum[a] || [0,0]
        // increment numerator if over_50k
        accum[a][0] += parseInt(curr.over_50k) ? 1 : 0
        // increment denominator
        accum[a][1]++
        return accum
    }

    // Applies data transformations:
    // 1. Uses a reducer function to transform csv-data object: [{education_level: "A", over_50k: "0"}, {education_level: "B", over_50k: "1"}]
    //      into bins by label, and counts of over_50k vs total counts: {A: [0,1], B: [1,1]} = intermediate result
    // 2. Loops through keys of intermediate result to construct an array: [{label: "A", value: 0}, {label: "B", value: 1}] = final result
    //      This form is easily digestible by d3.js to construct bar charts
    // 3. Sort the final result by descending value (percent over_50k)
    const transformData = (inputData, reducerFunc) => {
        let intermediate 
        let final = []
        if(inputData && inputData.length > 0){
            intermediate = inputData.reduce(reducerFunc, {})
            for(const k in intermediate){
                final.push({
                    label: k,
                    // intermediate[k][1] will never be 0. every key that is added to intermediate will be initialized with value = [0 or 1,1], and monotonically increases
                    value: intermediate[k][0] / intermediate[k][1]
                })
            }
            final = final.sort((a, b) => a.value < b.value)
        }
        return final
    }

    // when chartData changes, recalculate the d3-chart-svg. 
    // Clears and replaces existing chart.
    watch(chartData, () => {
        let cd = chartData.value
        if (cd && cd.length > 0){
            // Mostly taken from https://observablehq.com/@d3/horizontal-bar-chart/2?intent=fork
            const barHeight = 50
            const marginTop = 60
            const marginRight = 30
            const marginBottom = 10
            const marginLeft = 190
            const width = 928
            const rectHeight = Math.ceil((cd.length) * barHeight)
            const height = rectHeight + marginTop + marginBottom
            
            // scale functions
            const x = d3.scaleLinear()
                .domain([0, 1])
                .range([marginLeft, width - marginRight])

            const y = d3.scaleBand()
                .domain(cd.map(d => d.label))
                .rangeRound([marginTop, height - marginBottom])
                .padding(0.15)

            const colorMap = d3.scaleLinear()
                .domain([0, 1])
                .range(["#fb998e", "#99c0db"])

            const format = x.tickFormat(20, "%")

            // clear existing chart
            d3.select("#chart-container")
                .selectAll('*')
                .remove()

            // the auspicious svg
            const svg = d3.select("#chart-container")
                .append("svg")
                .attr("width", width)
                .attr("height", height)
                .attr("viewBox", [0, 0, width, height + 128])
                .attr("style", "max-width: 100%; height: auto; font: 10px sans-serif;")

            // axes
            svg.append("g")
                .attr("transform", `translate(0, ${height})`)
                .call(d3.axisBottom(x).ticks(width / 80, "%"))
               

            svg.append("g")
                .attr("transform", `translate(${marginLeft}, 0)`)
                .call(d3.axisLeft(y).tickSizeOuter(0).ticks(cd.length * 2))

            // gridlines (parallel to y-axis)
            svg.append("g")
                .attr("transform", `translate(0, ${height})`)
                .attr("style", "color:#e1e1e1;")
                .call(d3.axisBottom(x).tickSizeInner(-rectHeight).ticks(width / 80, ""))
                .call(g => g.select(".domain").remove())
                .call(g => g.selectAll("text").remove())
                

            // each data point gets a rectangle
            svg.append("g")
                .attr("fill", colorMap(1))
                .attr("fill-opacity", "0.7")
            .selectAll()
            .data(cd)
            .join("rect")
                .attr("x", x(0))
                .attr("y", (d) => y(d.label))
                .attr("width", (d) => x(d.value) - x(0) + 1)
                .attr("height", y.bandwidth())

            // complement rectangles for false result, rectangles should sum to 1
            svg.append("g")
                .attr("fill", colorMap(0))
                .attr("fill-opacity", "0.7")
            .selectAll()
            .data(cd)
            .join("rect")
                .attr("x", (d) => x(d.value))
                .attr("y", (d) => y(d.label))
                .attr("width", (d) => x(1 - d.value) - x(0) + 1)
                .attr("height", y.bandwidth())

            // label axes
            svg.append("text")
                .attr("transform", "rotate(-90)")
                .attr("x", - height / 2)
                .attr("y", 0)
                .attr("fill", "black")
                .style("text-anchor", "middle")
                .style("font-size", "20px")
                .text(activeAxis.value)

            svg.append("text")
                .attr("x", x(0.5))
                .attr("y", height * 1.1)
                .attr("fill", "black")
                .style("text-anchor", "middle")
                .style("font-size", "20px")
                .text("Percent Over 50K")

            // absolute legend (but is actually easily made dynamic)
            svg.append("g")
                .attr("stroke", "grey")
                .attr("fill-opacity", "0.3")
                .selectAll()
                .data([0, 1])
                .enter()
                .append("rect")
                    .attr("x", (d,i) => width - i*((40 + 5)*2))
                    .attr("y", 0)
                    .attr("width", 40)
                    .attr("height", 20)
                    .style("fill", d => colorMap(d))
                    

            svg.append("g")
                .style("font-size", "12px")
                .selectAll()
                .data(["False", "True"])
                .enter()
                .append("text")
                    .attr("x", (d,i) => width - i*((40+5)*2) + 5)
                    .attr("y", marginTop / 4)
                    .text(d => d)
                    

        }
    })
    
    onMounted(async () => {
        await fetchCSV()
    })
    
</script>

<template>
    <div>
        <h2>RTI Front-End Tech Test</h2>
        <div class="filters">
            <form>
                <!-- Better way to do this, only re-calc when form submit or something -->
                <input type="number" v-model="filterAgeGT"> &lt;= Age &lt;= <input type="number" v-model="filterAgeLT">
                <br>
                sex: <input type="text" v-model="filterSex">
                <br>
                Active Axis: <select v-model="activeAxis">
                    <option value="education_level">Education Level</option>
                    <option value="race">Race</option>
                </select>
            </form>
        </div>
        <div id="chart-container">
            
        </div>
        <p>{{ chartData && chartData.length > 0 ? chartData.slice(0,10) : "No Data Available" }}</p>
    </div>
</template>

<style>
</style>