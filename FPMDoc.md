# FootPrintMap

- 소속 : Jevinci
- 개발인원 : 2명(서버개발자, 클라이언트개발자)
- 개발기간 : 1년
- 개발 플렛폼 : iOS(iPhone)
- 개발 환경 : Swift3.0, MangoDB, SpringBoot, AWS
- 앱 개발 내용 : 여유 시간과 목적지로 지역의 명소를 추천하여 도보 여행 네비게이션을 제공합니다.
- 현재상태 : 재정문제로 개발중단

## 구조
![image info](./serverstructure.png)

## 기능
- 경로 데이터는 T Map API를 사용하여, 지도는 GoogleMap으로 표현
- 도보여행 네비게이션
- 인상깊은 여행 경로 minimap으로 저장
- 여러 사용자가 실시간으로 위치 공유

# 기능의 코드리뷰
### MODEL
경로 데이터 구조

<details><summary>CLICK ME</summary>
<p>

```swift
//  FPMRoute.swift
//  fpm
//
//  Created by Je.vinci.Inc on 2017. 10. 23..
//  Copyright © 2017년 Crycat. All rights reserved.
//

import Foundation
import GoogleMaps
import SwiftyJSON
import Darwin

enum FEATURETYPE :String{
    case POINT = "Point"
    case LINESTRING = "LineString"
}

enum POINTTYPE: String{
    case STARTPOINT = "SP"
    case ENDPOINT = "EP"
    case GP = "GP"
}

enum TURNTYPE: Int{
    case straight = 11
    case leftturn = 12
    case rightturn = 13
    case uturn = 14
    case eightturn = 16
    case tenturn = 17
    case twoturn = 18
    case fourturn = 19
    case wp = 184
    case wp1 = 185
    case wp2 = 186
    case wp3 = 187
    case wp4 = 188
    case wp5 = 189
    case overpass = 125
    case wtf = 126
    case enterstairs = 127
    case enterrunway = 128
    case enterstairsrunway = 129
    case start = 200
    case end = 201
    case crosswalk = 211
    case leftcrosswalk = 212
    case rightcrosswalk = 213
    case eightcrosswalk = 214
    case tencrosswalk = 215
    case twocrosswalk = 216
    case fourcrosswalk = 217
    case elevator = 218
    case tempstraight = 233
}

class Feature{
    var type: FEATURETYPE
    var index: Int
    var name: String?
    var description: String?
    init(featureType: FEATURETYPE, index: Int, description: String, name: String) {
        self.type = featureType
        self.index = index
        self.description = description
        self.name = name
    }
    func returnType() -> FEATURETYPE{
        return self.type
    }
}

class Point: Feature{
    var pointtype: POINTTYPE
    var pointindex: Int
    var turntype: TURNTYPE
    var coordinate: CLLocationCoordinate2D
    var nearpoiname: String?
    var nearpoix: String?
    var nearpoiy: String?
    var intersectionname: String?
    var facilitytype: String?
    var facilityname: String?
    var direction : String?
    var Ntotaldistance: Int?
    var totaltime: Int?
    init(featureType: FEATURETYPE, index: Int, pointType: POINTTYPE, pointindex: Int, turnType: TURNTYPE,coordinate: CLLocationCoordinate2D, description: String, name: String) {
        self.pointtype = pointType
        self.pointindex = pointindex
        self.turntype = turnType
        self.coordinate = coordinate
        super.init(featureType: featureType, index: index, description: description,name: name)
    }
    init(jsonObject: JSON){
        let property = jsonObject["properties"]
        let index = property["index"].intValue
        self.pointindex = property["pointIndex"].intValue
        self.pointtype = POINTTYPE(rawValue: property["pointType"].stringValue)!
        self.turntype = TURNTYPE(rawValue: property["turnType"].intValue)!
        let geometry = jsonObject["geometry"]
        let coordinate = geometry["coordinates"].arrayValue.map({$0.doubleValue})
        self.coordinate = CLLocationCoordinate2D(latitude: coordinate[1], longitude: coordinate[0])
        let description = property["description"].stringValue
        let name = property["name"].stringValue
        super.init(featureType: .POINT, index: index, description: description,name: name)
    }
}

class LineString: Feature{
    var lineindex: Int
    var distance: Int
    var time: Int
    var coordinates: Array<CLLocationCoordinate2D> = []
    var roadtype: Int?
    var categoryroadtype: Int?
    var facilitytype: String?
    var facilityname: String?
    init(featureType: FEATURETYPE, index: Int, lineindex: Int, distance: Int, time: Int, coordinates: Array<CLLocationCoordinate2D>, description: String, name: String) {
        self.lineindex = lineindex
        self.distance = distance
        self.time = time
        self.coordinates = coordinates
        super.init(featureType: featureType, index: index,description: description,name:name)
    }
    init(jsonObject: JSON){
        let property = jsonObject["properties"]
        let geometry = jsonObject["geometry"]
        let coordinatess = geometry["coordinates"].arrayValue
        for coordinates in coordinatess{
            let tempcoordinates = coordinates.arrayValue.map({$0.doubleValue})
            let tempcoordinate = CLLocationCoordinate2D(latitude: tempcoordinates[1], longitude: tempcoordinates[0])
            self.coordinates.append(tempcoordinate)
        }
        self.lineindex = property["lineIndex"].intValue
        self.distance = property["distance"].intValue
        self.time = property["time"].intValue
        let description = property["description"].stringValue
        let name = property["name"].stringValue
        super.init(featureType: .LINESTRING, index: property["index"].intValue, description: description,name: name)
    }
}

struct EncodedLine {
    var distance: Double
    var coordinate: CLLocationCoordinate2D
}

class Direction{
    var line: Array<EncodedLine> = []
    var sequence: Int
    var path: Array<Feature> = []
    init(sequence: Int, path: Array<Feature>) {
        self.sequence = sequence
        self.path = path
    }
    func getNearLineStringCoordinateIndex(location: CLLocationCoordinate2D, lastpointIdx: Int) -> Int?{
        var inlineIdx: Int?
        let _ = path.enumerated().map{ (index,value) in
            if value.returnType() == .LINESTRING{
                let line = value as! LineString
                if line.lineindex == lastpointIdx {
                    var shorter: Double = 0
                    let _ = line.coordinates.enumerated().map{ (index, cvalue) in
                        let distance = location.getRealMeterTo(to: cvalue)
                        if shorter > distance {
                            shorter = distance
                            inlineIdx = index
                        }
                    }
                }
            }
        }
        return inlineIdx
    }
    func getNearLineIndex(location: CLLocationCoordinate2D) -> Int{
        var Idx: Int = 0
        var comparedist = self.line[0].coordinate.getRealMeterTo(to: location)
        let _ = line.enumerated().map{ (index, value) in
            let distance = value.coordinate.getRealMeterTo(to: location)
            if comparedist > distance{
                Idx = index
                comparedist = distance
            }
        }
        return Idx
    }
    func getRealMeterToLineIdx(lineIdx: Int) -> Double{
        return self.line[lineIdx].distance
    }
    func getRealMeterFromLineIdx(lineIdx: Int) -> Double{
        let count = self.line.count
        let last = self.line[count - 1].distance
        return last - self.line[lineIdx].distance
    }
    func getRealMeterFromToLineIdx(from: Int, to: Int) -> Double{
        let end = self.line[to].distance
        let start = self.line[from].distance
        return end - start
    }
    func getPointArray() -> Array<Point>{
        var pointarray: Array<Point> = []
        for i in 0..<path.count{
            if path[i].returnType() == .POINT{
                pointarray.append(path[i] as! Point)
            }
        }
        return pointarray
    }

}
extension CLLocationCoordinate2D{
    func getRealMeterTo(to: CLLocationCoordinate2D) -> Double{
        let startcl = CLLocation(latitude: self.latitude, longitude: self.longitude)
        let endcl = CLLocation(latitude: to.latitude, longitude: to.longitude)
        return endcl.distance(from: startcl)
    }
}

extension LineString{
    func returnArrayCoordinate() -> Array<CLLocationCoordinate2D>{
        return self.coordinates
    }
}

extension Direction{
    func returnArrayCoordinate() -> Array<CLLocationCoordinate2D>{
        var result: Array<CLLocationCoordinate2D> = []
        for i in 0..<self.path.count{
            if path[i].returnType() == .LINESTRING{
                let linestring = path[i] as! LineString
                if i >= 2{
                    var line = linestring.returnArrayCoordinate()
                    line.remove(at: 0)
                    result.append(contentsOf: line)
                }else{
                    result.append(contentsOf: linestring.returnArrayCoordinate())
                }

            }
        }
        for i in 0..<result.count{
            if i == 0{
                let line = EncodedLine(distance: 0, coordinate: result[0])
                self.line.append(line)
            }else {
                let distance = result[i - 1].getRealMeterTo(to: result[i]) + self.line[i - 1].distance
                let line = EncodedLine(distance: distance, coordinate: result[i])
                self.line.append(line)
            }

        }
        return result
    }
    func returnArrayCoordinateFrom(lineIdx: Int, inlineIdx: Int) -> Array<CLLocationCoordinate2D>{
        var firstcoordinates: Array<CLLocationCoordinate2D> = []
        return []
    }
    func returnArrayCoordinateTo(lineIdx: Int, inlineIdx: Int) -> Array<CLLocationCoordinate2D>{
        return []
    }
}

extension FPMData{
    func returnArrayCoordinate() -> Array<CLLocationCoordinate2D>{
        var result: Array<CLLocationCoordinate2D> = []
        for i in 0..<self.directions.count{
            result.append(contentsOf: directions[i].returnArrayCoordinate())
        }
        return result
    }
}

extension GMSMutablePath{
    class func GMSMutablePathwithArrayCoordi(arraycoordinate: Array<CLLocationCoordinate2D>) -> GMSMutablePath{
        let path = GMSMutablePath()
        for i in 0..<arraycoordinate.count{
            path.add(arraycoordinate[i])
        }
        return path
    }
}
```

</p>
</details>
받은 장소 데이터를 T Map 로 변환하여 T Map API를 통해 경로 데이터를 받아옵니다.


### [LocalNotification, Circle을 통한 위치 확인 따른 네비게이션 안내]
### [UserData저장기능, MinimapCapture]
### [WebSocket을 통한 실시간 위치공유, Firebase를 통한 채팅, 파티 생성시 사용되는 APNS ]
### [WebSocket 채팅 코드내용]
