import SwiftUI
import Combine // Ditambahkan untuk memperbaiki error autoconnect()

struct ContentView: View {
    @State private var mediaURL: String = ""
    @State private var statusMessage: String = "Siap"
    @State private var isDownloading: Bool = false
    @State private var selectedFormat: String = "mp4"
    @State private var pixelOffset: CGFloat = -100
    @State private var frameIndex: Int = 0
    
    private let apiKey = "415c028989mshaaa2e45aecd1ab9p119029jsn9507a97e3973"
    private let apiHost = "instagram-downloader-download-instagram-videos-stories1.p.rapidapi.com"
    
    let timer = Timer.publish(every: 0.2, on: .main, in: .common).autoconnect()

    var body: some View {
        ZStack {
            Color.white.ignoresSafeArea()

            VStack(spacing: 20) {
                Text("Youtube Downloader")
                    .font(.title)
                    .fontWeight(.bold)
                    .foregroundColor(.red)
                    .padding(.top)

                TextField("Tempelkan tautan Instagram atau YouTube...", text: $mediaURL)
                    .textFieldStyle(RoundedBorderTextFieldStyle())
                    .padding(.horizontal)

                HStack(spacing: 10) {
                    FormatButton(title: "Video High (MP4)", isSelected: selectedFormat == "mp4") { selectedFormat = "mp4" }
                    FormatButton(title: "Audio Low (MP3)", isSelected: selectedFormat == "mp3") { selectedFormat = "mp3" }
                }
                .padding(.horizontal)

                Button(action: {
                    Task {
                        withAnimation { isDownloading = true }
                        if mediaURL.contains("youtu.be") || mediaURL.contains("youtube.com") {
                            await downloadYouTube()
                        } else {
                            await downloadInstagram()
                        }
                        withAnimation { isDownloading = false }
                    }
                }) {
                    Text(isDownloading ? "Memproses..." : "Unduh Sekarang")
                        .fontWeight(.bold)
                        .frame(maxWidth: .infinity)
                        .padding()
                        .background(isDownloading ? Color.gray : Color.red)
                        .foregroundColor(.white)
                        .cornerRadius(8)
                }
                .buttonStyle(PlainButtonStyle())
                .padding(.horizontal)
                .disabled(isDownloading || mediaURL.isEmpty)

                Spacer()

                if isDownloading {
                    VStack {
                        HStack(spacing: 15) {
                            PixelArtView(data: frameIndex == 0 ? kidFrame1 : kidFrame2)
                            PixelArtView(data: frameIndex == 0 ? butterflyFrame1 : butterflyFrame2)
                        }
                        .offset(x: pixelOffset)
                        .onAppear {
                            withAnimation(Animation.linear(duration: 3).repeatForever(autoreverses: false)) {
                                pixelOffset = 120
                            }
                        }
                        .onReceive(timer) { _ in
                            frameIndex = (frameIndex + 1) % 2
                        }
                        
                        Text("Sedang mengonversi, mohon tunggu...")
                            .font(.headline)
                            .foregroundColor(.red)
                            .padding(.top, 15)
                    }
                } else {
                    Text(statusMessage)
                        .foregroundColor(.black)
                }

                Spacer()
            }
            .padding()
        }
        .frame(minWidth: 450, minHeight: 350)
    }

    private func cleanInstagramURL(_ urlString: String) -> String {
        if let questionMarkIndex = urlString.firstIndex(of: "?") { return String(urlString[..<questionMarkIndex]) }
        return urlString
    }

    private func downloadInstagram() async {
        statusMessage = "Menghubungi API..."
        let cleanURL = cleanInstagramURL(mediaURL)
        var components = URLComponents(string: "https://\(apiHost)/fetch")
        components?.queryItems = [URLQueryItem(name: "url", value: cleanURL)]
        guard let url = components?.url else { return }

        var request = URLRequest(url: url)
        request.setValue(apiKey, forHTTPHeaderField: "X-RapidAPI-Key")
        request.setValue(apiHost, forHTTPHeaderField: "X-RapidAPI-Host")

        do {
            let (data, _) = try await URLSession.shared.data(for: request)
            if let json = try JSONSerialization.jsonObject(with: data) as? [String: Any] {
                let linkToDownload = (selectedFormat == "jpeg") ? (json["thumbnail"] as? String) : (json["download_url"] as? String)
                if let link = linkToDownload, let finalURL = URL(string: link) {
                    let ext = selectedFormat == "jpeg" ? "jpg" : (selectedFormat == "mp3" ? "mp3" : "mp4")
                    try await downloadFile(from: finalURL, filename: "InstagramMedia.\(ext)")
                } else { statusMessage = "Gagal: Format tidak tersedia." }
            }
        } catch { statusMessage = "Error: \(error.localizedDescription)" }
    }

    private func downloadYouTube() async {
        statusMessage = "Mengunduh dari YouTube..."
        let urlString = "https://\(apiHost)/youtube/download"
        guard let url = URL(string: urlString) else { return }
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue(apiKey, forHTTPHeaderField: "X-RapidAPI-Key")
        request.setValue(apiHost, forHTTPHeaderField: "X-RapidAPI-Host")
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        
        let formatToSend = selectedFormat == "jpeg" ? "1080p" : (selectedFormat == "mp3" ? "mp3" : "1080p")
        let bodyData: [String: Any] = ["url": mediaURL, "format": formatToSend]
        
        do {
            request.httpBody = try JSONSerialization.data(withJSONObject: bodyData)
            let (data, _) = try await URLSession.shared.data(for: request)
            if let json = try JSONSerialization.jsonObject(with: data) as? [String: Any],
               let link = json["download_url"] as? String ?? json["url"] as? String, let finalURL = URL(string: link) {
                let ext = selectedFormat == "mp3" ? "mp3" : "mp4"
                try await downloadFile(from: finalURL, filename: "YouTubeMedia.\(ext)")
            } else { statusMessage = "Gagal mengambil tautan YouTube." }
        } catch { statusMessage = "Error: \(error.localizedDescription)" }
    }

    private func downloadFile(from url: URL, filename: String) async throws {
        let (tempLocalUrl, _) = try await URLSession.shared.download(from: url)
        let fileManager = FileManager.default
        let downloadsFolder = fileManager.urls(for: .downloadsDirectory, in: .userDomainMask).first!
        let destinationUrl = downloadsFolder.appendingPathComponent(filename)
        if fileManager.fileExists(atPath: destinationUrl.path) { try fileManager.removeItem(at: destinationUrl) }
        try fileManager.moveItem(at: tempLocalUrl, to: destinationUrl)
        statusMessage = "Selesai! Tersimpan: \(filename)"
    }
}

struct PixelArtView: View {
    let data: [[Int]]
    var body: some View {
        VStack(spacing: 1) {
            ForEach(0..<data.count, id: \.self) { row in
                HStack(spacing: 1) {
                    ForEach(0..<data[row].count, id: \.self) { col in
                        Rectangle()
                            .fill(data[row][col] == 1 ? Color.red : Color.clear)
                            .frame(width: 3, height: 3)
                    }
                }
            }
        }
    }
}

let kidFrame1 = [[0,1,1,0],[0,1,1,0],[0,1,1,0],[1,1,1,1],[0,1,1,0],[0,1,1,0],[1,0,0,1]]
let kidFrame2 = [[0,1,1,0],[0,1,1,0],[0,1,1,0],[1,1,1,1],[0,1,1,0],[0,1,1,0],[0,1,1,0]]
let butterflyFrame1 = [[1,0,1],[1,1,1],[0,1,0],[1,1,1],[1,0,1]]
let butterflyFrame2 = [[0,1,0],[1,1,1],[1,1,1],[1,1,1],[0,1,0]]

struct FormatButton: View {
    let title: String; let isSelected: Bool; let action: () -> Void
    var body: some View {
        Button(action: action) {
            Text(title).fontWeight(.semibold).frame(maxWidth: .infinity).padding(.vertical, 8)
                .background(isSelected ? Color.red : Color.white).foregroundColor(isSelected ? Color.white : Color.red)
                .cornerRadius(6).overlay(RoundedRectangle(cornerRadius: 6).stroke(Color.red, lineWidth: 1))
        }.buttonStyle(PlainButtonStyle())
    }
}
