//
// FrameWatch.swift
// Lightweight frame-based motion detector simulation.
// Creates synthetic frames (grayscale matrices), computes frame-diff and simple blob detection.
// Swift 5+
//

import Foundation

typealias Frame = [[UInt8]]

func generateStaticFrame(width: Int, height: Int, base: UInt8) -> Frame {
    return (0..<height).map { _ in Array(repeating: base, count: width) }
}

func addMovingSquare(to frame: inout Frame, x: Int, y: Int, size: Int, value: UInt8) {
    let h = frame.count; let w = frame[0].count
    for yy in max(0,y)..<min(h, y+size) {
        for xx in max(0,x)..<min(w, x+size) {
            frame[yy][xx] = value
        }
    }
}

func frameDifference(_ a: Frame, _ b: Frame) -> Frame {
    let h = a.count; let w = a[0].count
    var out = a
    for y in 0..<h {
        for x in 0..<w {
            out[y][x] = abs(Int(a[y][x]) - Int(b[y][x])).clampedToUInt8()
        }
    }
    return out
}

extension Int {
    func clampedToUInt8() -> UInt8 {
        if self < 0 { return 0 }
        if self > 255 { return 255 }
        return UInt8(self)
    }
}

func threshold(_ frame: Frame, t: UInt8) -> [[Bool]] {
    return frame.map { row in row.map { $0 >= t } }
}

// simple connected-component labeling (4-neigh)
func connectedComponents(binary: [[Bool]]) -> [ [ (Int,Int) ] ] {
    let h = binary.count; let w = binary[0].count
    var visited = Array(repeating: Array(repeating: false, count: w), count: h)
    var comps: [[(Int,Int)]] = []
    for y in 0..<h {
        for x in 0..<w {
            if binary[y][x] && !visited[y][x] {
                var stack = [(x,y)]
                var comp: [(Int,Int)] = []
                visited[y][x] = true
                while let (cx,cy) = stack.popLast() {
                    comp.append((cx,cy))
                    for (nx,ny) in [(cx+1,cy),(cx-1,cy),(cx,cy+1),(cx,cy-1)] {
                        if nx >= 0 && nx < w && ny >= 0 && ny < h && binary[ny][nx] && !visited[ny][nx] {
                            visited[ny][nx] = true
                            stack.append((nx,ny))
                        }
                    }
                }
                comps.append(comp)
            }
        }
    }
    return comps
}

// Demo: moving square detection
func demoFrameWatch() {
    print("=== FrameWatch Demo ===")
    let W = 64, H = 32
    var prev = generateStaticFrame(width: W, height: H, base: 50)
    var frame = prev
    var posX = 0
    for t in 0..<80 {
        frame = generateStaticFrame(width: W, height: H, base: 50)
        addMovingSquare(to: &frame, x: posX, y: 10, size: 6, value: 200)
        let diff = frameDifference(frame, prev)
        let bin = threshold(diff, t: 30)
        let comps = connectedComponents(binary: bin)
        if !comps.isEmpty {
            print("t=\(t): detected \(comps.count) blobs, sizes: \(comps.map { $0.count })")
        }
        prev = frame
        posX += 1
        if posX > W { posX = -6 }
    }
    print("Demo complete.")
}

demoFrameWatch()
