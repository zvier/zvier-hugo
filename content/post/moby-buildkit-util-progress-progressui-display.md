---
title: "moby buildkit util progress progressui display"
date: 2020-03-01T09:01:41+08:00
draft: true
categories: ["技术"]
tags: ["buidkit"]
---
闻说双溪春尚好，也拟泛轻舟。只恐双溪舴艋舟，载不动、许多愁。——李清照《武陵春·春晚》
<!--more-->
# 文件路径
{{< highlight go "linenos=inline" >}}
github.com/moby/buildkit/util/progress/progressui/display.go
{{< /highlight >}}

# DisplaySolveStatus
{{< highlight go "linenos=inline" >}}
// "github.com/containerd/console"
func DisplaySolveStatus(ctx context.Context, phase string, c console.Console, w io.Writer, ch chan *client.SolveStatus) error {
    // 如果c非ni，采用console模式
	modeConsole := c != nil

	disp := &display{c: c, phase: phase}
	printer := &textMux{w: w}

    // [+] Building 2.8s (3/3)
	if disp.phase == "" {
		disp.phase = "Building"
	}

	t := newTrace(w, modeConsole)

	tickerTimeout := 150 * time.Millisecond
	displayTimeout := 100 * time.Millisecond

	if v := os.Getenv("TTY_DISPLAY_RATE"); v != "" {
		if r, err := strconv.ParseInt(v, 10, 64); err == nil {
			tickerTimeout = time.Duration(r) * time.Millisecond
			displayTimeout = time.Duration(r) * time.Millisecond
		}
	}

	var done bool
	ticker := time.NewTicker(tickerTimeout)
	defer ticker.Stop()

    // "golang.org/x/time/rate"
	displayLimiter := rate.NewLimiter(rate.Every(displayTimeout), 1)

	var height int
	width, _ := disp.getSize()
	for {
		select {
		case <-ctx.Done():
			return ctx.Err()
		case <-ticker.C:
		case ss, ok := <-ch:
			if ok {
				t.update(ss, width)
			} else {
				done = true
			}
		}

		if modeConsole {
			width, height = disp.getSize()
            // 最终进入done并结束
			if done {
				disp.print(t.displayInfo(), width, height, true)
				t.printErrorLogs(c)
				return nil
			} else if displayLimiter.Allow() {
				ticker.Stop()
				ticker = time.NewTicker(tickerTimeout)
				disp.print(t.displayInfo(), width, height, false)
			}
		} else {
			if done || displayLimiter.Allow() {
				printer.print(t)
				if done {
					t.printErrorLogs(w)
					return nil
				}
				ticker.Stop()
				ticker = time.NewTicker(tickerTimeout)
			}
		}
	}
    return nil
}
{{< /highlight >}}

# 常量
{{< highlight go "linenos=inline" >}}
const termHeight = 6
const termPad = 10
{{< /highlight >}}

# displayInfo
{{< highlight go "linenos=inline" >}}
type displayInfo struct {
	startTime      time.Time
	jobs           []*job
	countTotal     int
	countCompleted int
}
{{< /highlight >}}

# job
{{< highlight go "linenos=inline" >}}
type job struct {
	startTime     *time.Time
	completedTime *time.Time
	name          string
	status        string
	hasError      bool
	isCanceled    bool
	vertex        *vertex
	showTerm      bool
}
{{< /highlight >}}

# trace
{{< highlight go "linenos=inline" >}}
type trace struct {
	w             io.Writer
	localTimeDiff time.Duration
	vertexes      []*vertex
    // byDigest记录了各个digest的完成情况信息
	byDigest      map[digest.Digest]*vertex
	nextIndex     int
	updates       map[digest.Digest]struct{}
	modeConsole   bool
}
{{< /highlight >}}

# vertex
{{< highlight go "linenos=inline" >}}
type vertex struct {
	*client.Vertex
	statuses []*status
	byID     map[string]*status
	indent   string
	index    int

	logs          [][]byte
	logsPartial   bool
	logsOffset    int
	prev          *client.Vertex
	events        []string
	lastBlockTime *time.Time
	count         int
	statusUpdates map[string]struct{}

	jobs      []*job
	jobCached bool

	term      *vt100.VT100
	termBytes int
	termCount int
}
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
func (v *vertex) update(c int) {
	if v.count == 0 {
		now := time.Now()
		v.lastBlockTime = &now
	}
	v.count += c
}
{{< /highlight >}}

# status
{{< highlight go "linenos=inline" >}}
type status struct {
	*client.VertexStatus
}
{{< /highlight >}}

# newTrace
{{< highlight go "linenos=inline" >}}
// digest "github.com/opencontainers/go-digest"
func newTrace(w io.Writer, modeConsole bool) *trace {
	return &trace{
		byDigest:    make(map[digest.Digest]*vertex),
		updates:     make(map[digest.Digest]struct{}),
		w:           w,
		modeConsole: modeConsole,
	}
}
{{< /highlight >}}

func (t *trace) triggerVertexEvent(v *client.Vertex) {
	if v.Started == nil {
		return
	}

	var old client.Vertex
	vtx := t.byDigest[v.Digest]
	if v := vtx.prev; v != nil {
		old = *v
	}

	changed := false
	if v.Digest != old.Digest {
		changed = true
	}
	if v.Name != old.Name {
		changed = true
	}
	if v.Started != old.Started {
		if v.Started != nil && old.Started == nil || !v.Started.Equal(*old.Started) {
			changed = true
		}
	}
	if v.Completed != old.Completed && v.Completed != nil {
		changed = true
	}
	if v.Cached != old.Cached {
		changed = true
	}
	if v.Error != old.Error {
		changed = true
	}

	if changed {
		vtx.update(1)
		t.updates[v.Digest] = struct{}{}
	}

	t.byDigest[v.Digest].prev = v
}

func (t *trace) update(s *client.SolveStatus, termWidth int) {
	for _, v := range s.Vertexes {
		prev, ok := t.byDigest[v.Digest]
		if !ok {
			t.nextIndex++
			t.byDigest[v.Digest] = &vertex{
				byID:          make(map[string]*status),
				statusUpdates: make(map[string]struct{}),
				index:         t.nextIndex,
			}
			if t.modeConsole {
				t.byDigest[v.Digest].term = vt100.NewVT100(termHeight, termWidth-termPad)
			}
		}
		t.triggerVertexEvent(v)
		if v.Started != nil && (prev == nil || prev.Started == nil) {
			if t.localTimeDiff == 0 {
				t.localTimeDiff = time.Since(*v.Started)
			}
			t.vertexes = append(t.vertexes, t.byDigest[v.Digest])
		}
		// allow a duplicate initial vertex that shouldn't reset state
		if !(prev != nil && prev.Started != nil && v.Started == nil) {
			t.byDigest[v.Digest].Vertex = v
		}
		t.byDigest[v.Digest].jobCached = false
	}
	for _, s := range s.Statuses {
		v, ok := t.byDigest[s.Vertex]
		if !ok {
			continue // shouldn't happen
		}
		v.jobCached = false
		prev, ok := v.byID[s.ID]
		if !ok {
			v.byID[s.ID] = &status{VertexStatus: s}
		}
		if s.Started != nil && (prev == nil || prev.Started == nil) {
			v.statuses = append(v.statuses, v.byID[s.ID])
		}
		v.byID[s.ID].VertexStatus = s
		v.statusUpdates[s.ID] = struct{}{}
		t.updates[v.Digest] = struct{}{}
		v.update(1)
	}
	for _, l := range s.Logs {
		v, ok := t.byDigest[l.Vertex]
		if !ok {
			continue // shouldn't happen
		}
		v.jobCached = false
		if v.term != nil {
			if v.term.Width != termWidth {
				v.term.Resize(termHeight, termWidth-termPad)
			}
			v.termBytes += len(l.Data)
			v.term.Write(l.Data) // error unhandled on purpose. don't trust vt100
		}
		i := 0
		complete := split(l.Data, byte('\n'), func(dt []byte) {
			if v.logsPartial && len(v.logs) != 0 && i == 0 {
				v.logs[len(v.logs)-1] = append(v.logs[len(v.logs)-1], dt...)
			} else {
				ts := time.Duration(0)
				if v.Started != nil {
					ts = l.Timestamp.Sub(*v.Started)
				}
				v.logs = append(v.logs, []byte(fmt.Sprintf("#%d %s %s", v.index, fmt.Sprintf("%#.4g", ts.Seconds())[:5], dt)))
			}
			i++
		})
		v.logsPartial = !complete
		t.updates[v.Digest] = struct{}{}
		v.update(1)
	}
}

func (t *trace) printErrorLogs(f io.Writer) {
	for _, v := range t.vertexes {
		if v.Error != "" && !strings.HasSuffix(v.Error, context.Canceled.Error()) {
			fmt.Fprintln(f, "------")
			fmt.Fprintf(f, " > %s:\n", v.Name)
			for _, l := range v.logs {
				f.Write(l)
				fmt.Fprintln(f)
			}
			fmt.Fprintln(f, "------")
		}
	}
}

# trace.displayInfo
{{< highlight go "linenos=inline" >}}
func (t *trace) displayInfo() (d displayInfo) {
	d.startTime = time.Now()
	if t.localTimeDiff != 0 {
		d.startTime = (*t.vertexes[0].Started).Add(t.localTimeDiff)
	}
	d.countTotal = len(t.byDigest)
	for _, v := range t.byDigest {
		if v.Completed != nil {
			d.countCompleted++
		}
	}

	for _, v := range t.vertexes {
		if v.jobCached {
			d.jobs = append(d.jobs, v.jobs...)
			continue
		}
		var jobs []*job
		j := &job{
			startTime:     addTime(v.Started, t.localTimeDiff),
			completedTime: addTime(v.Completed, t.localTimeDiff),
			name:          strings.Replace(v.Name, "\t", " ", -1),
			vertex:        v,
		}
		if v.Error != "" {
			if strings.HasSuffix(v.Error, context.Canceled.Error()) {
				j.isCanceled = true
				j.name = "CANCELED " + j.name
			} else {
				j.hasError = true
				j.name = "ERROR " + j.name
			}
		}
		if v.Cached {
			j.name = "CACHED " + j.name
		}
		j.name = v.indent + j.name
		jobs = append(jobs, j)
		for _, s := range v.statuses {
			j := &job{
				startTime:     addTime(s.Started, t.localTimeDiff),
				completedTime: addTime(s.Completed, t.localTimeDiff),
				name:          v.indent + "=> " + s.ID,
			}
			if s.Total != 0 {
				j.status = fmt.Sprintf("%.2f / %.2f", units.Bytes(s.Current), units.Bytes(s.Total))
			} else if s.Current != 0 {
				j.status = fmt.Sprintf("%.2f", units.Bytes(s.Current))
			}
			jobs = append(jobs, j)
		}
		d.jobs = append(d.jobs, jobs...)
		v.jobs = jobs
		v.jobCached = true
	}

	return d
}
{{< /highlight >}}

func split(dt []byte, sep byte, fn func([]byte)) bool {
	if len(dt) == 0 {
		return false
	}
	for {
		if len(dt) == 0 {
			return true
		}
		idx := bytes.IndexByte(dt, sep)
		if idx == -1 {
			fn(dt)
			return false
		}
		fn(dt[:idx])
		dt = dt[idx+1:]
	}
}

func addTime(tm *time.Time, d time.Duration) *time.Time {
	if tm == nil {
		return nil
	}
	t := (*tm).Add(d)
	return &t
}

# display
{{< highlight go "linenos=inline" >}}
type display struct {
    // "github.com/containerd/console"
	c         console.Console
	phase     string
	lineCount int
	repeated  bool
}
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
func (disp *display) getSize() (int, int) {
	width := 80
	height := 10
	if disp.c != nil {
		size, err := disp.c.Size()
		if err == nil && size.Width > 0 && size.Height > 0 {
			width = int(size.Width)
			height = int(size.Height)
		}
	}
	return width, height
}
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
func setupTerminals(jobs []*job, height int, all bool) []*job {
	var candidates []*job
	numInUse := 0
	for _, j := range jobs {
		if j.vertex != nil && j.vertex.termBytes > 0 && j.completedTime == nil {
			candidates = append(candidates, j)
		}
		if j.completedTime == nil {
			numInUse++
		}
	}
	sort.Slice(candidates, func(i, j int) bool {
		idxI := candidates[i].vertex.termBytes + candidates[i].vertex.termCount*50
		idxJ := candidates[j].vertex.termBytes + candidates[j].vertex.termCount*50
		return idxI > idxJ
	})

	numFree := height - 2 - numInUse
	numToHide := 0
	termLimit := termHeight + 3

	for i := 0; numFree > termLimit && i < len(candidates); i++ {
		candidates[i].showTerm = true
		numToHide += candidates[i].vertex.term.UsedHeight()
		numFree -= termLimit
	}

	if !all {
		jobs = wrapHeight(jobs, height-2-numToHide)
	}

	return jobs
}
# display.print
{{< highlight go "linenos=inline" >}}
func (disp *display) print(d displayInfo, width, height int, all bool) {
	// this output is inspired by Buck
	d.jobs = setupTerminals(d.jobs, height, all)
	b := aec.EmptyBuilder
	for i := 0; i <= disp.lineCount; i++ {
		b = b.Up(1)
	}
	if !disp.repeated {
		b = b.Down(1)
	}
	disp.repeated = true
	fmt.Fprint(disp.c, b.Column(0).ANSI)

	statusStr := ""
	if d.countCompleted > 0 && d.countCompleted == d.countTotal && all {
		statusStr = "FINISHED"
	}

	fmt.Fprint(disp.c, aec.Hide)
	defer fmt.Fprint(disp.c, aec.Show)

    // [+] Building 2.3s (2/3) -> d.countCompleted, d.countTotal
	out := fmt.Sprintf("[+] %s %.1fs (%d/%d) %s", disp.phase, time.Since(d.startTime).Seconds(), d.countCompleted, d.countTotal, statusStr)
	out = align(out, "", width)
	fmt.Fprintln(disp.c, out)
	lineCount := 0
	for _, j := range d.jobs {
		endTime := time.Now()
		if j.completedTime != nil {
			endTime = *j.completedTime
		}
		if j.startTime == nil {
			continue
		}
		dt := endTime.Sub(*j.startTime).Seconds()
		if dt < 0.05 {
			dt = 0
		}
		pfx := " => "
		timer := fmt.Sprintf(" %3.1fs\n", dt)
		status := j.status
		showStatus := false

		left := width - len(pfx) - len(timer) - 1
		if status != "" {
			if left+len(status) > 20 {
				showStatus = true
				left -= len(status) + 1
			}
		}
		if left < 12 { // too small screen to show progress
			continue
		}
		name := j.name
		if len(name) > left {
			name = name[:left]
		}

		out := pfx + name
		if showStatus {
			out += " " + status
		}

		out = align(out, timer, width)
		if j.completedTime != nil {
			color := aec.BlueF
			if j.isCanceled {
				color = aec.YellowF
			} else if j.hasError {
				color = aec.RedF
			}
			out = aec.Apply(out, color)
		}
		fmt.Fprint(disp.c, out)
		lineCount++
		if j.showTerm {
			term := j.vertex.term
			term.Resize(termHeight, width-termPad)
			for _, l := range term.Content {
				if !isEmpty(l) {
					out := aec.Apply(fmt.Sprintf(" => => # %s\n", string(l)), aec.Faint)
					fmt.Fprint(disp.c, out)
					lineCount++
				}
			}
			j.vertex.termCount++
			j.showTerm = false
		}
	}
	// override previous content
	if diff := disp.lineCount - lineCount; diff > 0 {
		for i := 0; i < diff; i++ {
			fmt.Fprintln(disp.c, strings.Repeat(" ", width))
		}
		fmt.Fprint(disp.c, aec.EmptyBuilder.Up(uint(diff)).Column(0).ANSI)
	}
	disp.lineCount = lineCount
}
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
func isEmpty(l []rune) bool {
	for _, r := range l {
		if r != ' ' {
			return false
		}
	}
	return true
}

func align(l, r string, w int) string {
	return fmt.Sprintf("%-[2]*[1]s %[3]s", l, w-len(r)-1, r)
}

func wrapHeight(j []*job, limit int) []*job {
	var wrapped []*job
	wrapped = append(wrapped, j...)
	if len(j) > limit {
		wrapped = wrapped[len(j)-limit:]

		// wrap things around if incomplete jobs were cut
		var invisible []*job
		for _, j := range j[:len(j)-limit] {
			if j.completedTime == nil {
				invisible = append(invisible, j)
			}
		}

		if l := len(invisible); l > 0 {
			rewrapped := make([]*job, 0, len(wrapped))
			for _, j := range wrapped {
				if j.completedTime == nil || l <= 0 {
					rewrapped = append(rewrapped, j)
				}
				l--
			}
			freespace := len(wrapped) - len(rewrapped)
			wrapped = append(invisible[len(invisible)-freespace:], rewrapped...)
		}
	}
	return wrapped
}
{{< /highlight >}}

# rate.NewLimiter
NewLimiter returns a new Limiter that allows events up to rate r and permits bursts of at most b tokens.
{{< highlight go "linenos=inline" >}}
func NewLimiter(r Limit, b int) *Limiter
{{< /highlight >}}

# rate.Limit
Limit defines the maximum frequency of some events. Limit is represented as number of events per second. A zero Limit allows no events.
{{< highlight go "linenos=inline" >}}
type Limit float64
{{< /highlight >}}

# Limiter.Allow
Allow is shorthand for AllowN(time.Now(), 1).
{{< highlight go "linenos=inline" >}}
func (lim *Limiter) Allow() bool
{{< /highlight >}}

# Limiter.AllowN
AllowN reports whether n events may happen at time now. Use this method if you intend to drop / skip events that exceed the rate limit. Otherwise use Reserve or Wait.
{{< highlight go "linenos=inline" >}}
func (lim *Limiter) AllowN(now time.Time, n int) bool
{{< /highlight >}}
